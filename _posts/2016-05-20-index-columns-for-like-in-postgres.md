---
layout: post
title: Index Columns for `LIKE` in PostgreSQL
category: sql
tags: sql postgresql databases performance
description: Adding indexes for (i)LIKE searches in PostgreSQL
feature-img: images/sql-background.png
comments: true
---

Recently I wanted to add basic text search to an application I as working on. It took me a while to figure out the right way to index columns for `LIKE` lookups, especially for indexing compound columns. Here's how I did it.

## Introduction

Search is often an integral part of any web app, but it's also one of the parts that can cause performance problems. Users expect search to fast and to be accurate. Tools like [Elastic Search](https://www.elastic.co/products/elasticsearch) or [Solr](http://lucene.apache.org/solr/) are great at providing quick intelligent searches on large datasets. 

However, we don't always need such substantial dependencies in an app. Quite often, structuring and indexing the database correctly can keep your queries nice and fast.

## I ❤ Postgres

The more I work with PostgreSQL the more it impresses me. Whenever I need something from it, it's usually already there, I just cant always find how to do it. Indexing columns for `LIKE` queries was perfect example of this.

<blockquote class="twitter-tweet" data-lang="en">
  <p lang="en" dir="ltr">
    I&#39;m pretty sure Postgres has already solved most of my problems, I just haven&#39;t made it to that part of the documentation yet.
  </p>&mdash; Paul D. (@piisalie) 
  <a href="https://twitter.com/piisalie/status/708320089280352257">March 11, 2016</a>
</blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

## Implementation

I've got a very simple `users` database table populated with 1 million rows.

```sql
                                     Table "public.users"
   Column   |            Type             |                     Modifiers
------------+-----------------------------+----------------------------------------------------
 id         | integer                     | not null default nextval('users_id_seq'::regclass)
 username   | character varying           |
 first_name | character varying           |
 last_name  | character varying           |
```


### Without Any Index

Doing a quick case-insensitive `ILIKE` search with no matches is pretty slow:

```sql
$> EXPLAIN ANALYSE SELECT COUNT(*) FROM users WHERE username ILIKE '%foo%';
                                                  QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=21927.24..21927.25 rows=1 width=0) (actual time=737.523..737.523 rows=1 loops=1)
   ->  Seq Scan on users  (cost=0.00..21927.00 rows=96 width=0) (actual time=737.520..737.520 rows=0 loops=1)
         Filter: ((username)::text ~~* '%foo%'::text)
         Rows Removed by Filter: 1000000
 Planning time: 0.373 ms
 Execution time: 737.593 ms
(6 rows)

Time: 738.404 ms
```
### Regular index

You might think that indexing this column with a standard `btree` index would help this search - it doesn't. As you can see below, the query is just as slow. It's doing a full table scan and ignoring the index completely. 

The new index would be used if we're doing a comparison search such as `WHERE username = 'foo'`, but not if we're doing a partial match using `LIKE` or `ILIKE`:

```sql
$> CREATE INDEX idx_users_username ON users (username);
Time: 15987.751 ms
$> EXPLAIN ANALYSE SELECT COUNT(*) FROM users WHERE username ILIKE '%foo%';

                                                  QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=21927.24..21927.25 rows=1 width=0) (actual time=752.271..752.271 rows=1 loops=1)
   ->  Seq Scan on users  (cost=0.00..21927.00 rows=96 width=0) (actual time=752.268..752.268 rows=0 loops=1)
         Filter: ((username)::text ~~* '%foo%'::text)
         Rows Removed by Filter: 1000000
 Planning time: 0.599 ms
 Execution time: 752.318 ms
(6 rows)

Time: 753.251 ms
```

### Enter Postgres pg_trgm

> The pg_trgm module provides functions and operators for determining the similarity of ASCII alphanumeric text based on trigram matching, as well as index operator classes that support fast searching for similar strings.

Postgres uses trigrams to break down strings into smaller chunks and index them efficiently. The [pg_trgm](http://www.postgresql.org/docs/9.5/static/pgtrgm.html) module supports `GIST` or `GIN` indexes and as of Postgres version 9.1 these indexes support `LIKE`/`ILIKE` queries.

To use the `pg_trm` module, you need to enable the extension and create the index passing in the default `gin_trgm_ops`:

```sql
$> CREATE EXTENSION pg_trgm;
Time: 42.206 ms

$> CREATE INDEX trgm_idx_users_username ON users USING gin (username gin_trgm_ops);
Time: 7082.474 ms
```

Now running the same query takes only a fraction of the time. Just 2ms instead of 753ms! We can see from the query plan that it's using the new `trgm_idx_users_username` index we created.

```sql
$> EXPLAIN ANALYSE SELECT COUNT(*) FROM users WHERE username ILIKE '%foo%';

                                                               QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=369.12..369.13 rows=1 width=0) (actual time=0.030..0.030 rows=1 loops=1)
   ->  Bitmap Heap Scan on users  (cost=12.75..368.88 rows=96 width=0) (actual time=0.026..0.026 rows=0 loops=1)
         Recheck Cond: ((username)::text ~~* '%foo%'::text)
         ->  Bitmap Index Scan on trgm_idx_users_username  (cost=0.00..12.72 rows=96 width=0) (actual time=0.024..0.024 rows=0 loops=1)
               Index Cond: ((username)::text ~~* '%foo%'::text)
 Planning time: 0.636 ms
 Execution time: 0.095 ms
(7 rows)

Time: 2.333 ms

```
### Compound columns

In the table above we have users with a `first_name` and `last_name`.

Let's say we can search in our app for users by name - If I enter in 'John Doe' into my search field I would expect it to return a user named 'John Doe' (assuming it exists).

To do this we need to use both `first_name` and `last_name` columns and concatenate them in the query:

```sql
SELECT * FROM users WHERE first_name || ' ' || last_name ILIKE '%John Doe%'
```
If we create two new indexes on both `first_name` and `last_name` columns like we did for the `username` they won't be used by this query. The query is using a compound column so indexes on the individual columns won't help.

```sql
$> CREATE INDEX trgm_idx_users_first ON users USING gin (first_name gin_trgm_ops);
Time: 4577.637 ms
$> CREATE INDEX trgm_idx_users_last ON users USING gin (last_name gin_trgm_ops);
Time: 4770.507 ms
                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=27027.00..27027.01 rows=1 width=0) (actual time=1025.543..1025.544 rows=1 loops=1)
   ->  Seq Scan on users  (cost=0.00..26927.00 rows=40000 width=0) (actual time=1025.539..1025.539 rows=0 loops=1)
         Filter: ((((first_name)::text || ' '::text) || (last_name)::text) ~~* '%foo%'::text)
         Rows Removed by Filter: 1000000
 Planning time: 0.273 ms
 Execution time: 1025.591 ms
(6 rows)

Time: 1027.547 ms
```
Instead we need to create the index on the compound column we're using in the query.

```
$> CREATE INDEX index_users_full_name
             ON users using gin ((first_name || ' ' || last_name) gin_trgm_ops);

```

Running the query again we can see the compound index being used we're back to lightning fast speeds.

```
EXPLAIN ANALYSE SELECT COUNT(*) FROM users WHERE first_name || ' ' || last_name ILIKE '%foo%';
                                                                QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=10605.00..10605.01 rows=1 width=0) (actual time=0.020..0.020 rows=1 loops=1)
   ->  Bitmap Heap Scan on users  (cost=378.01..10505.00 rows=40000 width=0) (actual time=0.018..0.018 rows=0 loops=1)
         Recheck Cond: ((((first_name)::text || ' '::text) || (last_name)::text) ~~* '%foo%'::text)
         ->  Bitmap Index Scan on index_users_full_name  (cost=0.00..368.01 rows=40000 width=0) (actual time=0.016..0.016 rows=0 loops=1)
               Index Cond: ((((first_name)::text || ' '::text) || (last_name)::text) ~~* '%foo%'::text)
 Planning time: 0.338 ms
 Execution time: 0.080 ms
(7 rows)

Time: 0.975 ms
```

#### Why we're not using concat_ws

When I initially built the query I used `concat_ws` (concat with separator) [string function](http://www.postgresql.org/docs/9.5/static/functions-string.html) to join `first_name` and `last_name`.

```sql
concat_ws(' ', first_name, last_name)
```
However, `concat_ws` cannot be used in an index as it's not immutable. See this [discussion](http://www.postgresql.org/message-id/3361.1410026366@sss.pgh.pa.us) for more information on that.

### Conclusion

Once again digging into the rich features of Posgtres helps to stave of adding dependencies on other search tools. The [pg_trgm](http://www.postgresql.org/docs/9.5/static/pgtrgm.html) module provides far more than just better indexing for text columns. It provides rich fuzzy searching capabilities and can handle full-text serach. This [blog post](http://blog.lostpropertyhq.com/postgres-full-text-search-is-good-enough/)gives some great ideas of what is capable with your application database.

Thanks to [this post](https://www.depesz.com/2011/02/19/waiting-for-9-1-faster-likeilike/) by [@the_depesz](https://twitter.com/the_depesz) for pointing me in the right direction.
