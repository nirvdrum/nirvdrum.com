---
layout: post
title: Composite Index Problem with PostgreSQL and Rails 2.x
author: Kevin Menard
---

Introduction
------------

Recently I ran into an issue with using composite indices in PostgreSQL and Rails 2.3.2.  I only managed to
catch the problem by using the [shoulda should\_have\_index](http://dev.thoughtbot.com/shoulda/classes/Shoulda/ActiveRecord/Macros.html#M000057) macro.
This macro asserts that an index appears on a list of columns.  Since it is a list, the order of the columns is
in fact significant.

Problem
--------

The problem is that given a table with the following definition:

{% highlight ruby %}
  create_table "video_games", :force => true do |t|
    t.string   "asin"
    t.integer  "user_id", :null => false
  end
{% endhighlight %}

and the following migration:

{% highlight ruby %}
  add_index :video_games, [:user_id, :asin], :unique => true
{% endhighlight %}

the schema dumper for the ActiveRecord PostgreSQL adapter may actually produce the following in schema.rb:

{% highlight ruby %}
  add_index "video_games", ["asin", "user_id"],
    :name => "index_video_games_on_user_id_and_asin",
    :unique => true
{% endhighlight %}

The distinction here is subtle, but important.  In the migration, I declared the index should be on the tuple ``(user_id, asin)`` and the schema dumper in turn generated code that would add a tuple on ``(asin, user_id)``.

The issue was with the way that the adapter was fetching the index data.  It issued a query against PostgreSQL's
maintenance tables to reconstruct the index pseudo-DDL statement.  The query used in Rails 2.3.2 is:

{% highlight sql %}
  SELECT distinct i.relname, d.indisunique, a.attname
     FROM pg_class t, pg_class i, pg_index d, pg_attribute a
   WHERE i.relkind = 'i'
     AND d.indexrelid = i.oid
     AND d.indisprimary = 'f'
     AND t.oid = d.indrelid
     AND t.relname = '#{table_name}'
     AND i.relnamespace IN (SELECT oid FROM pg_namespace WHERE nspname IN (#{schemas}) )
     AND a.attrelid = t.oid
     AND ( d.indkey[0]=a.attnum OR d.indkey[1]=a.attnum
        OR d.indkey[2]=a.attnum OR d.indkey[3]=a.attnum
        OR d.indkey[4]=a.attnum OR d.indkey[5]=a.attnum
        OR d.indkey[6]=a.attnum OR d.indkey[7]=a.attnum
        OR d.indkey[8]=a.attnum OR d.indkey[9]=a.attnum )
  ORDER BY i.relname
{% endhighlight %}

There's a lot going on there that may be hard to follow.  The query returns the index name (``i.relname``),
a Boolean indicating whether or not the index is unique (``d.indisunique``), and a member column of the index (``a.attname``).  For
composite indices, there are multiple rows, one for each member column.

The important thing to note is that ``d.indkey`` is a PostgreSQL array type (``int2vector``) that contains a list
of column positions for member columns of the index.  As can be seen by the query, there is no explicit ordering
of the ``a.attname``, so PostgreSQL is free to return the rows in any order it wishes.  In PostgreSQL 8.3, this ordering
appears to be attribute's positional index, in ascending order.  Please not that I have not consulted the
PostgreSQL source to verify this.  Suffice it to say, the returned ordering should not be relied upon and is
not guaranteed to match the order in ``d.indkey``.  The problem is that the schema dumper did in fact rely on this order.

As an aside, there is another problem with this query.  It will only index 10 elements of the d.indkey array, leading
to a ceiling of 10 columns per index.  This is a Rails-imposed limit.  As of at least PostgreSQL 7.4, that limit is
32 by default and can be configured higher at compile-time.


Resolution
----------

Both issues were fixed as of April 21, 2009 with the closing of [Rails ticket #2515](https://rails.lighthouseapp.com/projects/8994-ruby-on-rails/tickets/2515), nearly 3.5 years after the problem
was first introduced on September 23, 2005.  Interestingly, the problem was reported by three different parties
in April 2009.  Between the time I came across it and then eventually came up with a fix and filed a ticket, someone else reported the issue and fixed it.  So, that's how I ended up with this analysis of a problem that in the end I didn't have to solve.

Interestingly, the issue shows up with ``rake db:test:load`` but not ``rake db:test:clone_structure`` because the
former uses the ActiveRecord PostgreSQL adapter's implementation of schema dumping and loading, whereas the latter
uses the pg\_dump tool to create a DDL file.  ``rake db:test:prepare`` does a ``clone_structure`` followed by a ``load``,
which yields a test database that does not match the correct one used in development.
