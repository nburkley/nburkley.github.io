---
layout: post
title: Changing Primary Keys in Ecto
category: elixir
tags: elixir phoenix ecto postgresql
description: Changing the Primary Key of an existing database table with Ecto.
feature-img: images/sql-background.png
comments: true
---

Recently I needed to change the primary key on an existing database table in a Phoenix project. While it sounds straight forward, there were a few hoops to jump through. I put together a writeup on how I went about it.


## What we have

Let's take an example of a books table. We've been using an [ISBN](https://en.wikipedia.org/wiki/International_Standard_Book_Number) string value for the `id` primary key.

The table has the following structure:

```sql
+-------------+-----------------------------+-------------+
| Column      | Type                        | Modifiers   |
|-------------+-----------------------------+-------------|
| id          | character varying(255)      |  not null   |
| title       | character varying(255)      |  not null   |
| page_count  | integer                     |             |
| inserted_at | timestamp without time zone |  not null   |
| updated_at  | timestamp without time zone |  not null   |
+-------------+-----------------------------+-------------+
Indexes:
    "books_pkey" PRIMARY KEY, btree (id)
```

## What we want

Rather than using a ISBN string as the primary key, we're going use "serial" datatype.

We'll also copy the existing `id` value to a new `isbn` column which we'll create on our books table.

---
#### Aside: Serial Data types
A serial is an auto-incrementing, non null numeric value. This is what phoenix uses by default as a primary for database tables.

More details on the serial datatypes can be found in the PostgreSQL and MySQL docs:

* [PostgreSQL Serial Datatype](https://www.postgresql.org/docs/9.1/static/datatype-numeric.html#DATATYPE-SERIAL)
* [MySQL Numeric Types](https://dev.mysql.com/doc/refman/5.7/en/numeric-type-overview.html)

---

After making our changes, our books table will have the following structure:

```sql
+-------------+-----------------------------+-----------------------------------------------------+
| Column      | Type                        | Modifiers                                           |
|-------------+-----------------------------+-----------------------------------------------------|
| id          | bigint                      |  not null default nextval('books_id_seq'::regclass) |
| title       | character varying(255)      |  not nul                                            |
| page_count  | integer                     |                                                     |
| isbn        | character varying(255)      |  not null                                           |
| inserted_at | timestamp without time zone |  not null                                           |
| updated_at  | timestamp without time zone |  not null                                           |
+-------------+-----------------------------+-----------------------------------------------------+
Indexes:
    "books_pkey" PRIMARY KEY, btree (id)
```

## How we get there

We'll create a separate `up` and `down` function for our migration, our migration is to complicated for a `change` to infer how it should be rolled back.

Here's what we need in our `up/0` function:

### 1. Create some new columns

```elixir
alter table(:books) do
  add(:new_primary_id, :serial)
  add(:isbn, :string)
end

flush()
```

First we add a new column `"new_primary_id"` that will become our primary key. Using the `serial` datatype, it will automatically be populated with unique ascending numbers

Add an `"isbn"` column that we'll copy our existing`"id"` values into. While we want a `not null` constraint on the column, we can't add that until it's fully populated or we'll be in violation of that constraint.

Use the [flush()](https://hexdocs.pm/ecto/Ecto.Migration.html#flush/0) function to ensure the new columns are added immediately so they can be referenced later in the migration. Thanks to this [great article from HashRocket](https://hashrocket.com/blog/posts/ecto-migrations-simple-to-complex#transitioning-a-column) for helping me figure that one out.


### 2. Copy over the existing id values

```elixir
import Ecto.Query, only: [from: 2]

from(b in "books", update: [set: [isbn: b.id]])
|> MyApp.Repo.update_all([])
```

Next we copy over the values from our old `"id"` to the `"isbn"` column we've just created.

*Note*: We're using a [Schemaless Migration](http://blog.plataformatec.com.br/2016/05/ectos-insert_all-and-schemaless-queries/) so we're referencing our database directly, not our Ecto Schemas. This ensures our migration isn't affected by any future changes to our Ecto.Schemas.


### 3. Swap over to our new primary key

```elixir
alter table(:books) do
  remove(:id)
  modify(:new_primary_id, :integer, primary_key: true)
  modify(:isbn, :string, null: false)
end
```

We remove the old primary key `"id"` column. This also drops the primary key constraint associated with this column. 

We now set our `"new_primary_id"` as the primary key for books. A table cannot have two primary keys (we're not talking about composite keys here), so we have to drop the old one before we can set the new one.

Now that we've populated the `"isbn"` column in step #3, we can add the `not null` constraint on the column.


### 4. Rename our new primary key

Our column `"new_primary_id"` is now set up and populated so we can rename it to `"id"`, getting us back to the standard primary key convention in Ecto.

```elixir
rename(table(:books), :new_primary_id, to: :id)
```

### 5. Add an index

Even though the ISBN value is no longer our primary key, it's still likely the table will be queried by ISBN, so let's add an index on that column.

```elixir
create(index(:books, :isbn))
```

## Do it all again, backwards

For our `down/0` method, we basically need to run through all of the above steps, in reverse order. Doing so will ensure that running `mix ecto.rollback` will revert our table back to it's original state.

## All together now

As I mentioned at the start, swapping primary keys is something that appears to be a simple task at first, but because relational databases have such tight constraints (and rightly so), we have to do a bit of work to achieve our goal.

Running `mix ecto.migrate` will:
* move the ISBN values from `"id"` to a new `"isbn"` column
* change the "`id`" column type to an auto-incrementing numeric and populate all the values

Here's what our final migration looks like. 

```elixir
defmodule MyApp.Repo.Migrations.AddNewBooksPrimaryKey do
  use Ecto.Migration
  alias MyApp.Repo

  import Ecto.Query, only: [from: 2]

  def up do

    # 1. Create new columns
    # add new_primary_id to books which will take over as primary key
    # add isbn to store existing id
    alter table(:books) do
      add(:new_primary_id, :serial)
      add(:isbn, :string)
    end

    # ensure the new columns are added
    flush()

    # 2. Copy over the existing isbn values
    from(b in "books", update: [set: [isbn: b.id]])
    |> Repo.update_all([])


    # 3. Swap over to our new primary key
    alter table(:books) do
      remove(:id)
      modify(:new_primary_id, :integer, primary_key: true)
      modify(:isbn, :string, null: false)
    end

    # 4. Rename our new primary key
    rename(table(:books), :new_primary_id, to: :id)

    # 5. Add an index to isbn
    create(index(:books, :isbn))
  end

  def down do
    # Add back our old primary key
    alter table(:books) do
      add(:old_primary_id, :string)
    end

    # ensure the new column is added
    flush()

    # populate the old_primary_id column
    from(c in "books", update: [set: [old_primary_id: c.isbn]])
    |> Repo.update_all([])

    # swap back the primary key
    alter table(:books) do
      remove(:id)
      remove(:isbn)
      modify(:old_primary_id, :string, primary_key: true)
    end

    # rename the primary key
    rename(table(:books), :old_primary_id, to: :id)
  end
end
```
