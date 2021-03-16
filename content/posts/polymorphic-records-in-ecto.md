---
title: "Polymorphic Records in Ecto"
date: 2021-03-15T12:25:10-07:00
draft: false
---

Parameterized types landed in Ecto [3.5](https://twitter.com/josevalim/status/1300448536681684992?ref_src=twsrc%5Etfw%7Ctwcamp%5Etweetembed%7Ctwterm%5E1300448536681684992%7Ctwgr%5E%7Ctwcon%5Es1_c10&ref_url=https%3A%2F%2Fpublish.twitter.com%2F%3Fquery%3Dhttps3A2F2Ftwitter.com2Fjosevalim2Fstatus2F1300448536681684992widget%3DTweet) and with them some new magical powers for us alchemists. My favorite being polymorphic types. What are polymorphic types you ask?

How often has this happened to you? You are using a SQL database and need to model a collection of conceptually similar items that all have a slightly different shape? Let's give a concrete example. You are implementing an activity feed for the hottest Fintech startup in the valley. In this feed you want to show a list of user transactions - things such as deposits, transfers, card transactions, and interest earned events.

It's tempting to want to group these things together and call them "transactions". While they represent a financial transaction, each one has unique details to it. On a card transaction you would expect to see merchant details. On interest earned events there would be, well, interest earned amounts.

## Goals and Requirements

Before we dive into the code lets fully flesh out the requirements so that we can build our solution around them. 

* We are using a SQL database that supports maps - I'll be assuming Postgres
* Querying the transaction feed is performant and scalable
* Each transaction line item has its own enforceable schema
* New activity items can be added to the feed without needing changes to the database or hurting performance
* Use the Ecto patterns we already know and love
* Use Elixir - No trade offs here :)

> There are always tradeoffs and alternatives - I'll discuss those briefly at the end.

## Table for one please

To start our adventure we will need to create a schema for our database table. This one table will house all of our transaction records. For all the fields we can confidently say will be on every record we will make columns. For all other fields we will use our polymorphic type.

We don't want to be too aggressive on deciding what fields are shared between all records since we want to support unknown transaction types down the road. It feels safe to assume every transaction would have an amount so lets just go with that. 

Next, we need a way to have a field that can be any one of our transaction types. We could just define it as a `:map` and call it a day but there are some issues with that. For one, our unique transaction types are not just loose key value pairs. They probably have some sort of schema, just not one we can enforce at the database level. Instead we will define a polymorphic type.

{{< code language="elixir" >}}
defmodule TransactionFeedItem do
  use Ecto.Schema
  alias Ecto.Changeset
  
  schema "transaction_feed_items" do
    # shared fields
    field :amount, :decimal
    # polymorphic field
    field :data, PolymorphicType, interest: InterestEarnedTransaction, card: CardTransaction
  end
  
  def changeset(%__MODULE__{} = transaction_feed_item, params \\ %{}) do
    transaction_feed_item
    |> cast(params, [:amount, :data])
    |> validate_required([:amount, :data])
  end
end
{{< /code >}}

We will jump to the implementation of `PolymorphicType` shortly. In the meantime we can soak in what an `Ecto.ParameterizedType` looks like. Parameterized types are a lot like other `Ecto.Type` you might have implemented before. The key difference is that they can accept options that alters their behaviour. In our case we can map a `type` of transaction to a custom `Ecto.Type` that handles it. Essentially saying that one of these types of `Ecto.Type` will reside here.

Lets first implement one of these custom types. I'll go with the `InterestEarnedTransaction`.

{{< code language="elixir" >}}
defmodule InterestEarnedTransaction do
  use Ecto.Type
  use Ecto.Schema
  alias Ecto.Changeset

  @primary_key false
  embedded_schema do
    field :earn_on, :utc_datetime
    field :type, Ecto.Enum, values: [:interest]
  end

  @impl Ecto.Type
  def type(), do: :map

  @impl Ecto.Type
  def cast(data) do
    %__MODULE__{}
    |> Changeset.cast(data, [:earn_date, :type])
    |> Changeset.put_change(:type, :earn)
    |> Changeset.validate_required([:earn_date])
    |> case do
      %Changeset{valid?: true} = changeset ->
        {:ok, Changeset.apply_changes(changeset)}

      _error_changeset ->
        :error
    end
  end

  @impl Ecto.Type
  def dump(%__MODULE__{} = module) do
    {:ok, Map.from_struct(module)}
  end

  @impl Ecto.Type
  def load(data) do
    cast(data)
  end
end
{{< /code >}}

> You would maybe want to include a `cast/1` implementation that could handle being passed the struct instead of a map - I'll leave that as an exercise to the reader.

To be a custom type `Ecto.Type` we have to implement the `type/0`, `cast/1` `dump/1`, and `load/1` callbacks. These callbacks are mostly for declaring what database type we map to, casting types and checking validity, mapping to a shape the database can handle, and how to load from the database type to our type. I chose to add an embedded_schema to the mix so that we can use all of the `Ecto.Changeset` functions to validate our custom type.

Now how do we wire it up so that we can pass this as a field to our `TransactionFeedItem`. Time for the polymorphic type.

## What we all came to see

Time to implement the parameterized type shown earlier that will give us a polymorphic field. A parameterized type looks a lot like other custom ecto types. The difference is that the options given to the type when used in the schema are passed into some of the callbacks. Also a new `init/1` callback is introduced. It gives us a hook to prepare our params into a shape that is easier to work with in the other callbacks. Our polymorphic type will operate as a high level type that will look into params passed to delegate to the correct type that handles it based on the `type` field.

{{< code language="elixir" >}}
defmodule PolymorphicType do
  use Ecto.ParameterizedType

  # what database type we are
  # since we need to store multiple values
  # we need to be a map - this becomes a jsonb column in Postgres
  @impl Ecto.ParameterizedType
  def type(_params), do: :map

  # prepare args for easier pattern matching
  @impl Ecto.ParameterizedType
  def init(opts) do
    Enum.into(opts, %{})
  end

  # define how we cast when given a struct
  # we can just delegate to that struct
  @impl Ecto.ParameterizedType
  def cast(struct, _types) when is_struct(struct) do
    struct.__struct__.cast(struct)
  end

  # define how we cast when given params
  # in this case we have to look up the type
  @impl Ecto.ParameterizedType
  def cast(data, types) do
    type = type_from(types, data)

    type.cast(data)
  end

  @impl Ecto.ParameterizedType
  def dump(struct, _dumper, _types) do
    struct.__struct__.dump(struct)
  end

  @impl Ecto.ParameterizedType
  def load(data, _loader, types) do
    type = type_from(types, data)
    type.load(data)
  end
  
  # Helpers for delegating types

  # look up responsible type for params
  # handle when params are atoms (most likely code is casting)
  defp type_from(types, %{type: type}) do
    Map.get(types, type)
  end

  # look up responsible type for params
  # handle when params is string keyed (most likely code is loading from db)
  defp type_from(types, %{"type" => type}) when is_binary(type) do
    Map.get(types, String.to_existing_atom(type))
  end
end
{{< /code >}}

## With great power

Lets take it for a spin, this code is now possible.

{{< code language="elixir">}}
%TransactionFeedItem{amount: Decimal.new(100) }
|> TransactionFeedItem.changeset(%{data: %{type: :interest, earn_date: DateTime.utc_now()})
|> Repo.insert()
{{< /code >}}

In this example the `PolymorphicType` will be called to cast the params. It looks it up based on the `type` field and finds that the `InterestEarnedTransaction` type handles the `interest` type. It then delegates to that type. If all is well in the cast it will attempt to insert it into the database.

We don't just get this niceness when inserting into the database. Since it all works with Ecto types we can also query the database and have our type transformed.

{{< code language="elixir" >}}
iex> Repo.all(TransactionFeedItem)
[
  %TransactionFeedItem{
    mount: Decimal.new(100),
    data: %InterestEarnedTransaction{
      type: :interest,
      earn_date: %DateTime{}
  }
{{< /code >}}

Elixir you never cease to amaze me!

## Did we make it?

Lets loop back on the original goals we had set for ourselves to see how we measure up.

- [x] SQL is our datastore
- [x] Querying the transaction feed is performant and scalable
  - We only have to query one table rather than doing joins
  - We do need to make sure we have a good pagination strategy in place for this though
- [x] Each transaction line item has its own enforceable schema
- [x] New activity items can be added to the feed without needing changes to the database
  - Implement new custom `Ecto.Type`
  - Implement the callbacks
  - Add type as option to the `PolymorphicType`
- [x] Use the Ecto patterns we already know and love
- [x] In Elixir

---

## Trade offs

### Do you even SQL?
In my implementation I am clearly trying to bend SQL to work similar to how document databases naturally model things. When people say that document databases are schemaless they don't mean there is no schema. They mean that the schema is not enforced compile time but instead at runtime. In my example I am clearly trying to bypass SQL schemas and push the responsibility to the runtime. If you are in a position to just use a document database for this problem you will shed a lot of complexity.

### Performance

We are also moving some computation of parsing these schemas into our Elixir code. I personally think its worthwhile to take pressure of your database wherever possible, but your situation might be different.

### Share-ability

If the database is being shared and moved to other sources there is a shape mismatch between our database and our in memory structures.

### Querying

When you receive back a record from the database it will be perfectly parsed into your types, but when building queries that filter on these custom fields you will need to do json queries since the native type in the database are jsonb columns.
