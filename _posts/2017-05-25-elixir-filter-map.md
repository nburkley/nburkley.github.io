---
layout: post
title: Elixir's `Enum.filter_map`
category: elixir
tags: elixir filter_map, filter, map, enumerable
description: An dive into using elixir's Enum.filter_map function.
comments: true
---

When dealing with collections, I frequently find myself filtering and mapping a collection to extract the results I need. Elixir's `Enum` library has a great `filter_map/3` function to simplify this…

The general concept of `filter_map` is pretty straight forward, but the first time I came across it I didn't quite grok it straight away. I thought it may help others to explain in a bit more detail how to use it.

## Deprecated [update 12/08/17]

`Enum.filter_map/3` has been [deprecated](https://github.com/elixir-lang/elixir/blob/v1.5/CHANGELOG.md#4-deprecations) in Elixir 1.5:

> Deprecate Enum.filter_map/3 in favor of Enum.filter/2 + Enum.map/2 or for-comprehensions

I've added some 'Alternative Implementations' to the examples below using `Enum.filter/2` + `Enum.map/2`, or for-comprehensions.

## Definition

Here's the definition [from hexdocs](https://hexdocs.pm/elixir/Enum.html#filter_map/3):

> `filter_map(enumerable, filter, mapper)`
>
> Filters the enumerable and maps its elements in one pass

And the example:

```elixir
iex> Enum.filter_map([1, 2, 3], fn(x) -> rem(x, 2) == 0 end, &(&1 * 2))
[4]
```

## Explanation

The example uses the `&` shorthand for an anonymous function, but when you're new to the language it can be a little confusing. Let's re-write it without the shorthand:

```elixir
Enum.filter_map(
  [1, 2, 3],                                    # enumerable
  fn(value) -> rem(value, 2) == 0 end,          # filter
  fn(filtered_value) -> filtered_value * 2 end  # mapper
)
```
### Here's what's happening:

#### 1. _enumerable_
This is our enumerable collection that we begin with, in this case it's `[1,2,3]`.

#### 2. _filter_
This is a function defining how we want to filter the values. The filter function should return a truthy/non-truthy (everything is truthy except `nil` and `false`) result for each value. Only values that return a truthy value for the filter function will pass through.

In this case we want values from the collection that are divisible by 2. This function will be applied to each value in our enumerable.

```elixir
fn(value) -> rem(value, 2) == 0 end
``` 

The only value in our enumerable that will return `true` for this function is `2`, so our filter returns `[2]`.

#### 3. _mapper_

This is a function that will be applied to each value of the filter results, and returns the results in a list.

```elixir
fn(filtered_value) -> filtered_value * 2 end
``` 

This function just multiples each value by 2, so it transforms the filter results `[2]` to return `[4]`.

## Alternative Implementations

Here's how the example above could be implemented using `Enum.filter/2` + `Enum.map/2`:

```elixir
[1,2,3]
|> Enum.filter(fn(value) -> rem(value, 2) == 0 end)
|> Enum.map(fn(filtered_value) -> filtered_value * 2 end)
```

Or it could also be written using a `for` comprehension:

```elixir
for value <- [1, 2, 3], rem(value, 2) == 0, do: value * 2
```

## Further example

Here's a slightly more complicated example. We've got a list of Structs containing user information:"

```elixir
  users = [
    %{ id: 1, full_name: "John Doe", username: "jdoe", email: "jdoe@email.com"},
    %{ id: 2, full_name: "Mary Doe", email: "mdoe@email.com"},
    %{ id: 3, username: "dfaker", email: "dfaker@email.com"},
    %{ id: 4, email: "noname@email.com"}
  ]
```

Let's say we want a list of tuples in the form `{id, name}` from our list of users, and we've got the following business logic:
- `name` can be the users `full_name` or `username` - if they have both, prioritize the `full_name`
- we want the names in uppercase
- if a user has no `full_name` or `username` we return nothing for that user

There many different ways we to do this, but one way, using `Enum.filter_map` could look like this:

```elixir
defmodule FilterMapExample do
  def filter_users(users) do
    Enum.filter_map(
      users,
      fn(user) -> has_name?(user) end,
      fn(user) -> filterd_data(user) end
    )
  end

  defp has_name?(%{full_name: _full_name}), do: true
  defp has_name?(%{username: _username}), do: true
  defp has_name?(_), do: nil

  defp filterd_data(%{id: id, full_name: name}), do: {id, String.upcase(name)}
  defp filterd_data(%{id: id, username: name}), do: {id, String.upcase(name)}
end
```

We've pulled out our filter and mapper functions into private methods `has_name?` and `filtered_data` and use pattern matching to return the correct result from these private methods.

Using the example to filter the users collection we get a list of tuples containing only users with a full_name/username in uppercase:

```elixir
iex> FilterMapExample.filter_users(users)
[{1, "JOHN DOE"}, {2, "MARY DOE"}, {3, "DFAKER"}]
```

#### Simplifying `filter_user`

Using the `&` shorthand and `|>`, the `filter_users` methods could be simplified:

```elixir
def filter_users(users) do
  users
  |> Enum.filter_map(&has_name?/1, &filterd_data/1)
end
```
---

## Alternative Implementations

Here's how the `filter_users/1` above could be implemented using `Enum.filter/2` + `Enum.map/2`:

```elixir
def filter_users(users) do
  users
  |> Enum.filter(&has_name?/1)
  |> Enum.map(&filterd_data/1)
end
```

Or it could also be written using a `for` comprehension:

```elixir
def filter_users(users) do
  for user <- users, has_name?(user), do: filterd_data(user)
end
```

---

It's a handy method for working with collections, but going forward you should use `Enum.filter/2` + `Enum.map/2` or for-comprehensions instead. Either way, I hope this helps you understand how to use `filter_map` a little better!
