---
title: "Polymorphic Records in Ecto"
date: 2021-03-01T12:25:10-07:00
draft: true
---

How often has this happened to you? You are using a SQL database and need to model a collection of conceptually similar items that all have a slightly different shape? Let's give a concrete example. You are implementing an activity feed for the hottest Fintech startup in the valley. In this feed you want to show a list of user transactions - things such as deposits, transfers, card transactions, and interest earned events.

It's tempting to want to group these things together and call them "transactions". While they represent a financial transaction, each one has unique details to it. On a card transaction you would expect to see merchant details. On interest earned events there would be, well, interest earned percentages.

## Goals and Requirements

Before we dive into the code lets fully flesh out the requirements so that we can build our solution around them. 

* SQL is our datastore
* Querying the transaction feed is performant and scalable
* Each transaction line item has its own enforceable schema
* New activity items can be added to the feed without needing changes to the database
* Use the Ecto patterns we already know and love
* Use Elixir - No trade offs here :)

> There are always trade offs and other ways of implementing this. These will be discussed at the end of the article.

## Table for one please

To start our adventure we will need to create a schema for our database table. This one table is where all of our records will be stored. For all the fields we can confidently say will be on every record we will make columns. For all other fields we will have to store them as a json map.

We don't want to be too aggressive on deciding what fields are shared between all records since we want to support unknown transaction types down the road. It feels safe to assume every transaction would have an amount so lets just go with that. We also want to make sure that we can define what all the possible types are. For this we will define an enum to hold that value.

{{< code language="elixir" >}}
defmodule TransactionFeedItem do
  use Ecto.Schema
  alias Ecto.Changeset
  
  schema "transaction_feed_items" do
    # define our transaction types
    field :type, Ecto.Enum, values: [:deposit, :interest_earned]
    # shared fields
    field :amount, :decimal
    # unique to every transaction type
    field :data, :map
  end
end
{{< /code >}}

The keen eyed among you might notice that with this schema alone we can't enforce the `data` field since its just a `map` type. Ecto will happily accept any map there. Hold tight we will solve that in a moment. The next thing we can tackle is a changeset for the schema.

{{< code language="elixir" >}}
def changeset(transaction_feed_item, params \\ %{}) do
  transaction_feed_item
  |> cast(params, [:type, :amount, :data])
  |> validate_required([:type, :amount, :data])
end
{{< /code >}}

## Embedded schemas to the rescue

Here is where things get interesting. Ecto has [embedded](https://hexdocs.pm/ecto/Ecto.Schema.html#embedded_schema/1) schemas. These are schemas that are not backed by an actual database table. They are great for using all of Ecto's changeset functionality over html forms, external apis, and as we are about to see Polymorphic records.

Let's go with defining the deposit transaction feed item.

{{< code language="elixir" >}}
defmodule DepositTransactionFeedItem do
  use Ecto.Schema
  alias Ecto.Changeset
  
  embedded_schema do
    # define common fields
    field :type, Ecto.Enum, values: [:deposit, :interest_earned]
    field :amount, :decimal
    # now some custom fields unique to this type
    field :initiated_at, :utc_datetime
    field :completed_at, :utc_datetime
  end
end
{{< /code >}}

And an accompanying changeset for it.

{{< code language="elixir" >}}
def changeset(deposit_transaction, params \\ %{}) do
  deposit_transaction
  |> cast(params, [:amount, :initiated_at, :completed_at])
  # we know this is the deposit type every time
  # so we can remove the need to pass it in
  |> put_change(:type, :deposit)
  |> validate_required([:type, :amount, :initiated_at, :completed_at])
end
{{< /code >}}

> There is some serious code duplication here but we will come back with a fix for that

So now we have a database table and schema along with an embedded_schema for the unique transaction type. How do we get to where we can persist our unique transaction?

## Transaction feed item behaviour

For all of our unique transaction types we can make them implement a behaviour that can go from the database table schema to them and vice versa. Lets define that behaviour.

{{< code language="elixir" >}}
defmodule TransactionFeedItem do
  use Ecto.Schema
  alias Ecto.Changeset
  
  @doc """
  Go from feed item to a transaction type
  """
  @callback from_feed_item(%TransactionFeedItem{}) :: map()
  
  @doc """
  Go from a unique transaction type to a feed item
  """
  @callback to_feed_item(map) :: %TransactionFeedItem{}
  
  # snip ...
  # schema and changeset code
  # snip ...
end
{{< /code >}}

Now we can implement this behaviour for our deposit transaction.

{{< code language="elixir" >}}
defmodule DepositTransactionFeedItem do
  use Ecto.Schema
  alias Ecto.Changeset
  
  @behaviour TransactionFeedItem
  
  @impl TransactionFeedItem
  def from_feed_item(%TransactionFeedItem{} = feed_item) do
    # grab unique params from data
    params = %{
      initiated_at: feed_item.data.initiated_at,
      completed_at: feed_item.data.completed_at
    }
    
    # we can use our own changeset to help build our record back up
    %__MODULE__{
      id: feed_item.id,
      amount: feed_item.amount
    }
    |> changeset(params)
    |> apply_changes()
  end
  
  @impl TransactionFeedItem
  def to_feed_item(%__MODULE__{} = deposit_transaction) do
    %TransactionFeedItem{
      id: deposit_transaction.id,
      amount: deposit_transaction.amount,
      # nest our unique fields under the json map key
      data: %{
        initiated_at: deposit_transaction.initiated_at,
        completed_at: deposit_transaction.completed_at
      }
    }
  end
  
  # snip ...
  # schema and changeset code
  # snip ...
end
{{< /code >}}

## Final steps

Before we seal the deal and glue this all together let's recap what we have so far. We created a schema for our database table. Next we created an embedded_schema for our unique transaction type. Then we created a behaviour that describes how to go from unique records to the database table and vice versa. Lastly, we implemented the behaviour for our deposit transaction.

All of the pieces are in place now, but a good high level API can help glue it all together. To work with `TransactionFeedItem`s lets create a context module. With this module we can create an API to create, fetch, and update our unique records. Let's work our way through the functions one at a time, starting with the simplest.


### Getting a unique transaction by type

Passing in the type and an id we can convert to our type

{{< code language="elixir" >}}
defmodule TransactionFeedItems do
  @doc """
  Get a TransactionFeedItem by id and type
  """
  def get(type, id) do
    case Repo.get(TransactionFeedItem, id) do
      nil ->
        nil
        
      %TransactionFeedItem{} = item ->
        # use behaviour to get in correct shape
        type.from_feed_item(item)
    end
  end
end
{{< /code >}}

### Inserting a unique transaction


{{< code language="elixir" >}}
defmodule TransactionFeedItems do
  # snip ...
  # def get(type, id) do
  # snip ...
  
  # handle if given a %TransactionFeedItem{} based changeset
  def insert(%Changeset{valid?: true, data: %{__struct__: TransactionFeedItem} = changeset}) do
    Repo.insert(changeset)
  end
  
  # we were given a valid custom transaction type
  # time to do the behaviour to convert it to table schema
  def insert(%Changeset{valid?: true, data: data} = changeset) do
    changeset
    # give us the underlying struct
    |> Changeset.apply_changes()
    # convert it to table schema shape
    |> data.__struct__.to_feed_item()
    # run it through top level changeset just to double check implemtations
    |>TransactionFeedItem.changeset()
    # insert if valid
    |> Repo.insert()
  end
  
  # for all other cases give back changeset as an error
  def insert(invalid_changeset), do: {:error, invalid_changeset}
end
{{< /code >}}

### Updating a unique transaction

And lastly, the ability to modify a unique transaction. For this to work we send all the fields on the record as changes. This makes sense since most of the fields would end up in the `data` map and we have to replace the whole thing anyways.

{{< code language="elixir" >}}
defmodule TransactionFeedItems do
  # snip ...
  # def get(type, id) do
  # def insert(...) do
  # snip ...
  
  # handle if given a TransactionFeedItem to udpate
  def update(%Changeset{data: %{__struct__: TransactionFeedItem}} = changeset) do
    Repo.update(changeset)
  end
  
  def update(%Changeset{valid?: true, data: data} = changeset) do
    # take all of the fields as changes
    changes = changeset
    |> Changeset.apply_changes()
    |> data.__struct__.to_feed_item()
    |> Map.from_struct()
    
    %TransactionFeedItem{id: changes.id}
    |> TransactionFeedItem.changeset(changes)
    |> Repo.update()
  end
end
{{< /code >}}

## Extra steps

Whew that was a lot of details to get out. What we have above works but as I mentioned there is some code duplication that we can take care of to make this even easier to work with.

One of the issues we have is around defining the embedded_schema. They are responsible for defining all the shared fields as well as the ones they want to add. If down the line we need to add a new shared field you would need to go to all of their schemas and include that field. This could be especially bad given the `type` Enum might be defined all over the place. Also we are forcing them to verify in their changesets that the shared fields have been supplied. Surely we can provide more helpers to solve these issues.

Starting with creation of schemas we can use a simple macro to make defining them easier. Basically defining the shared fields for them and letting them add the custom fields they need.

{{< code language="elixir" >}}
defmodule TransactionFeedItem do
  defmacro new_transaction_item(do: block) do
    quote do
      use Ecto.Schema
      
      embedded_schema do
        # inject their custom fields
        unquote(block)
        
        # now add all shared fields
        field :type, Ecto.Enum, values: [:deposit, :interest_earned]
        field :amount, :decimal
      end
    end
  end
end
{{< /code >}}

This means we can now define our scheme in the deposit module as:

{{< code language="elixir" >}}
defmodule DepositTransactionFeedItem do
  alias Ecto.Changeset
  
  # get access to macro
  require TransactionFeedItem 
  
  TransactionFeedItem.new_transaction_item do
    field :initiated_at, :utc_datetime
    field :completed_at, :utc_datetime
  end
end
{{< /code >}}

Okay now for the changesets. We can create helper functions that check for only the required fields which our unique transactions can use in their changesets

{{< code language="elixir" >}}
defmodule TransactionFeedItem do
  def cast_shared(data, params \\ %{}) do
    cast(data, params, [:type, :amount])
  end
  
  def validate_required_shared(changeset) do
    validate_required(changeset, [:type, :amount])
  end
end
{{< /code >}}

This would make our `DepositTransactionFeedItem` changeset now be able to look like this.

{{< code language="elixir" >}}
defmodule DepositTransactionFeedItem do
  def changeset(deposit_transaction, params \\ %{}) do
    deposit_transaction
    # cast shared params
    |> TransactionFeedItem.cast_shared()
    # cast our unique params
    |> cast(params, [:initiated_at, :completed_at])
    |> put_change(:type, :deposit)
    # validate required shared
    |> TransactionFeedItem.validate_required_shared()
    # validate our unique fields
    |> validate_required([:initiated_at, :completed_at])
  end
end
{{< /code >}}

## Did we make it?

Lets loop back on the original goals we had set for ourselves to see how we measure up.

- [x] SQL is our datastore
- [x] Querying the transaction feed is performant and scalable
  - We only have to query one table rather than doing joins
  - We do need to make sure we have a good pagination strategy in place for this though
- [x] Each transaction line item has its own enforceable schema
- [x] New activity items can be added to the feed without needing changes to the database
  - Add your type to the `type` Enum
  - Implement the callbacks
  - No database changes needed
- [x] Use the Ecto patterns we already know and love
- [x] In Elixir - No trade offs here :)


---

## Trade offs

### Do you even SQL?
In my implementation I am clearly trying to bend SQL to work similar to how document databases naturally model things. When people say that document databases are schemaless they don't mean there is no schema. They mean that the schema is not enforced compile time but instead at runtime. In my example I am clearly trying to bypass SQL schemas and push the responsibility to the runtime. If you are in a position to just use a document database for this problem you will shed a lot of complexity.

### Performance

We are also moving some computation of parsing these schemas into our Elixir code. I personally think its worthwhile to take pressure of your database wherever possible, but your situation might be different.

### Share-ability

If the database is being shared and moved to other sources there is a shape mismatch between our database and our in memory structures.
