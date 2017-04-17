+++
date = "2017-04-12T17:00:30-07:00"
title = "Window Functions in Redshift"
draft = false

+++

One of the coolest things I learned about in my Redshift journey has been Window Functions.

Although Window functions aren't a novel feature and exists in other SQL databases, they are a really powerful tool to have in your analysis toolbelt and fits in really well with Redshift.

Like the name suggests, Window Functions let you operate on frame or 'window' of data and return a value for each row in that result set.

An easy way to understand this is to think about a spreadsheet. Cells in a spreadsheet can reference arbitrary ranges of other cells relative to themselves and use their values as input to functions.

Window functions work in much the same way, and thus can be super powerful for the same kind of operations you can perform in a spreadsheet and even beyond that.

In this post, I'll walk you through some of the Window Functions that I've found the most useful and hopefully help you gain a better intuition on how they work in general.  

## Looking ahead or behind

The most basic window functions help you access data from 'previous' or 'following' rows, much like you can do in a spreadsheet.

Two of the most useful window functions are the `lead` and `lag` functions.

Let's say you have a simple table of timeseries data, for instance the closing balance of your stock market account per day.

Now let's say for each day you would like to calculate your loss/gain over the previous day's balance.

One way to do this would be something like this:

```sql
select
  b.date
  , b.balance
  , b.balance - previous_day.balance as gain
from balances b
left join balances previous_day on dateadd(day, -1, b.date) = previous_day.date
order by date
```

The `lag` window function allow us to simplify things, without having to do a self-join.

```sql
select
  date
  , balance
  , balance - lag(balance,1) over (
      order by date
  ) as gain
from balances
order by date
```

The `lag` function allows us to grab the value of a column in a 'previous' row (`lead ` does the opposite, by accessing the next row). We can specify the ordering of rows within the window function in the `over(order by date)` clause.

One interesting thing to note is that the window function does not rely on the ordering of the query itself. If we were to order differently in the outer query, the `lag` function would still calculate `gain` correctly.

Another pair of functions that are pretty useful for looking ahead or backwards are `first_value` and `last_value`.

Say you always want to calculate the difference of your daily closing balance with the first balance you had when opening the account, you can use the `first_value` window function to do this:

```sql
select
  date
  , balance
  , balance - first_value(balance) over (
      order by date
      rows unbounded preceding
  ) as total_gain
from balances
order by date
```

This time, we also include a frame clause: `rows unbounded preceding`. This tells our window function to use all the rows preceding the current row, after sorting on the `date` field.

## Aggregations

Window functions also lets you do more complex aggregations on frames of data.

Let's say you also have `trades` table where you keep track of all your stock trading. It's pretty simple and it has `ts` (timestamp), `symbol` and `profit` columns.

Now let's say you want to list out all my trades over time, with a running total of your profit per symbol.

We can use the `sum` window function to do this:

```sql
select
  ts
  , symbol
  , profit
  , sum(profit) over (
      partition by symbol
      order by ts
      rows unbounded preceding
   ) total_profit_for_symbol
from trades
order by date
```

As you can see, we're using the `sum` aggregation as a window function to calculate our running total. This is different from a normal `sum` aggregation you would do in Redshift, because you don't need to add a `group by` clause, but instead you're adding an `over` clause.

In the `over` clause, we're still sorting by date and we also have a frame clause.

We also include a `partition by symbol` clause. This will essentially 'filter' our aggregation to only include rows that have the same value for `symbol` as our current row.

## Ranking and numbering

Aside from aggregation, Window Functions can also do ranking or numbering of rows. This might not seem especially interesting at first, but once you have this tool at your disposal you'll find lots of uses for it.

For instance, let's say you have duplicated data that you want to filter out, but your table has no primary key to that you can use to figure out which records are duplicated.

Let's say I had a bug in my code and I accidentally adding a bunch of duplicate trades to my `trades` table and wanted to write a query to clean things up.

This is where the `row_number()` function can come in very handy.

```sql
with dupe_trades as (
   select
       *
       , row_number() over (
           partition by ts, symbol, profit
       )
   from trades
   order by ts
)
select * from dupe_trades
where row_number = 1
```

Since we're partitioning by all the columns in the table, any set of rows with duplicate values will have a `row_number` column, starting at `1` to `n`, where `n` is the number of duplicates.

To filter out duplicates, we can just take all the rows with a `row_number` of `1`. This will just give us back the first row out of the set of duplicates.

You might be wondering why I used a [CTE](https://docs.aws.amazon.com/redshift/latest/dg/r_WITH_clause.html) here. Well, one gotcha of Window Functions is that they can't be used in a `where` clause (the window function is applied to the result set after the where clause is run).

By wrapping the Window Function in a CTE with the `with` clause, we can tell Redshift that we want to apply the Window Function before filtering.

If we wanted to see how many duplicates we have on average by symbol, we can do this:

```sql
with with_row_number as (
   select
       *
       , row_number() over (
           partition by ts, symbol, profit
      )
   from trades
   order by ts
),
with_number_of_duplicates as (
   select
       *
       , (max(row_number) over (
           partition by ts,symbol,profit
         ) -1)  as number_of_duplicates
   from with_row_number
   group by 1,2,3,4
)
select
   symbol
   , sum(number_of_duplicates) as number_of_duplicates_for_symbol
from with_number_of_duplicates
group by 1
order by 2 desc
```

Here we're making use of the `max` aggregation as a window function to get the number of duplicates per set of duplicate values, together with our trick of grouping by all the other columns.

## Avoiding correlated subqueries

Window Functions can be used to avoid correlated subqueries.

A subquery is correlated when it references values from the outer query. Correlated subqueries need to be evaluated for each row of the outer query and can thus be slow.

To illustrate, let's return back to our `trades` table. Let's say I want to list all the trades, and see not only the profit for that trade but how it compares to the average profit for that symbol.

Here's how you would do it with a correlated subquery.

```sql
select
  ts
  , symbol
  , profit
  (
    select
      avg(profit)
    from trades t2
    where t2.symbol = t1.symbol
   ) average_profit_for_symbol
from trades t1
order by date
```

Since the subquery references the `symbol` column in the outer query, it will need to be evaluated for each row.

If we had a lot of data, this could be slow. Here's how we can implement this using the `avg` window function.

```sql
select
  date
  , symbol
  , profit
  , avg(profit) over (
      partition by symbol
  ) as average_profit_for_symbol
from trades
order by date
```

Ah much cleaner!

## Doing Statistics

Window functions are super useful for calculating standard summary statistics.

Let's revisit the query I wrote earlier to compare the profit on a trade to the average profit for trades on that symbol. Let's say instead of the mean, I want to use the median.

To do this, I could use the `median` window function.

```sql
select
  date
  , symbol
  , profit
  , median(profit) over (
      partition by symbol
  ) as median_profit_for_symbol
from trades
order by date
```

We could also do more interesting stats like the percentiles and standard deviations.

```sql
select
  date
  , symbol
  , profit
  , median(profit) over (
      partition by symbol
  ) as median_profit_for_symbol
  , stddev_pop(profit) over (
    partition by symbol
  ) as profit_std_dev_for_symbol
  , ntile(4) over (
    partition by symbol
    order by profit
  ) as profit_ntile_for_symbol
from trades
order by date
```

## Aggregating lists

Finally, you can use window functions as a way to concatenate a list of column values into a single value. The `listagg` function will group and order a list of rows and then concatenate it's values into a single string.

Let's say for a particular trading day we want to not only see the closing balance, but also a list of symbols that we've trading that day in a comma separated string.

```sql
select
  date
  , balance
  , balance - first_value(balance) over (
      order by b.date
      rows unbounded preceding
  ) as total_gain
  , listagg(symbol, ', ') symbols_traded
from balances
left join trades on date(balances.date) = date(trades.ts)
group by 1,2
order by date
```

One thing to be careful of is the the total size of the resulting string cannot be bigger than a `varchar(max)` which is 65535 characters.

## That's a wrap!

That concludes our short tour of window functions in Redshift. My hope is that I have convinced you that they can be a powerful addition to your analysis tool belt and  this gives you a good start to go explore more on your own and learn more about all the [other window functions](https://docs.aws.amazon.com/redshift/latest/dg/c_Window_functions.html) that you can use in Redshift!
