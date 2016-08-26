---
layout: post
title: Preloading aliases in Elixir `iex` shell
category: elixir
tags: elixir phoenix
description: Load up your `iex` shell with your aliases already set
feature-img: images/iex.png
comments: true
---

I've been working on my first [Phoenix](http://www.phoenixframework.org/) project recently. I found it very annoying to have to add an aliases for every model I want to interact with in `iex`. There's an easier way…

## What I was doing

Let's say I want to view all the `Article`s in my database. I would load up `iex` by running '`iex -S mix`' and then run:

```elixir
alias MyApp.Repo
alias MyApp.Article
Repo.all(Article)
```


## Using an `.iex.exs` file

I recently discovered that you can use a local `.iex.exs` project to set your aliases. For more on using an `.iex.exs` file, see the [iex docs](http://elixir-lang.org/docs/v1.1/iex/IEx.html).

Create an `.iex.exs` file in the root of your project and add the aliases you need:


```elixir
alias MyApp.Repo
alias MyApp.User
alias MyApp.Team
alias MyApp.Article
```

Now when you load up your `iex` shell using '`iex -S mix`' your aliases will already be set. To view all the articles we can just run:

```elixir
Repo.all(Article)
```

### Shorthand

As of Elixir 1.2 you can alias multiple submodules in one call

```elixir
alias MyApp.{Repo, User, Team, Article}
```


