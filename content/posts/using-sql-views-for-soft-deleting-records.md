+++
title = "Using SQL views for soft deleting records"
author = ["Orlando Collins"]
date = 2022-10-21
tags = ["sql"]
categories = ["programming"]
draft = false
+++

## Using SQL views to manage soft deletes {#using-sql-views-to-manage-soft-deletes}

> **Soft Deletion:** A way of marking a record as deleted actually deleting anything and losing data

When casually messing aroung with SQL you have all the CRUD operations available
available to you. Have fun with this power while it lasts because in real world
applications you will rarely get the chance to delete anything. More often than
not you will hear messaging that deleting is an a [anti-pattern](https:www.infoq.com/news/2009/09/Do-Not-Delete-Data/).


### Strategies {#strategies}

I have seen a few different strategies to cope with soft deletes. Here are the
major ones and my proposition on a strategy worth considering.


#### Flag and filter soft deletes {#flag-and-filter-soft-deletes}

The most common soft delete strategy is to add a `deleted_at` column. Using this
column we can filter out soft deleted records and maintain information about
when the deletion occured.

Done. Problem solved. Its no wonder this is the prevailing pattern.

From the implementation side things are simple,
but in usage this pattern is frought with chances to shoot ourselves in the
foot.

One of the major annoyances of working with tables containing flagged soft
deleted records is having to continually remember to filter out records when
forming new queries. This problem also shows up when any table associates to a
table with soft deleted records. It's not hard to imagine a time comes where we
mess up and show deleted records by accident.


#### Move strategy {#move-strategy}

Another strategy involves using two different tables; one for active records and
the other for the deletes. Soft deleting a record now means deleting it from the
active records table and placing it into the table containing the deleted records.

The benefits are that we no longer need to filter out anything. Just point all
your queries at the active table.

I see two major negatives with this approach. First, now two different schemas
need to be maintained for the same record. Second, and perhaps the bigger issue,
is that since a real delete operation happens we run the risk of messing up and
deleting records forever. Yikes, its no wonder people play it safe and stick
with the `flag` strategy.


#### Defining a view {#defining-a-view}

Now for the main event, and my personal favorite strategy (and criminally
underused SQL feature) - defining views to hide deleted records. The strategy
starts out similar to the `flag` strategy where we add a `deleted_at` column to
the table. The new step is defining a view to only maintain the active records.

This example view will be a ficticious ticketing system where we never want to
delete tickets.

```sql
CREATE VIEW active_tickets AS
SELECT *
FROM tickets
WHERE deleted_at IS NOT NULL
```

Now we get the best of both worlds. No risk of losing data at and we get the
feeling of a separate table `active_tickets` to query. No more forgetting to
filter.

Happy SQLing!
