---
title: Guards in Typespecs
date: 2022-08-13
draft: false
toc: false
summary: Using guard clauses in Typespecs to progressively define the types
tags: [elixir, til]
categories: [programming]
---

When defining a typespec for an elixir function it is possible to define it
using a guard clause.

```elixir
@spec my_function(arg) :: arg when arg: atom()
def my_function(arg), do: arg
```

At first glance this doesn't seem all that useful, at least it wasn't for me,
but then I realized there is more to the story. Guards in typespecs also allow
specifying any type as a variable and later defining what they are. I love this
for the situation where you are defining a type inline that has some inner
types that make it nested and noisy to spell out from top to bottom.

For example here is a fake API client which takes query parameters. Notice the use
of `list_params`, `sort_order`, and `unix_timestamp` variables.

```elixir
@spec list(list_params) :: {:ok, [Todo.t()]},
  when list_params: %{
    optional(:limit) => non_neg_integer(),
    optional(:sort) => sort_order,
    optional(:after) => unix_timestamp,
    optional(:before) => unix_timestamp
  },
  sort_order: :ASC | :DESC,
  unix_timestamp: non_neg_integer()
def list(params) do
  case Todos.Client.list(params) do
    # ...
  end
end
```

I appreciate being able to skip over some of the types and define them later.
My eyes have an easier time finding the important type definition that I am
looking for when it is broken out this way.

## References

* [Elixir Typespec Docs](https://hexdocs.pm/elixir/typespecs.html)

