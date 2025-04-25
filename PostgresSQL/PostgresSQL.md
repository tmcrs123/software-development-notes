
# Never forget:

LEFT JOIN -> "I want all records from the left table and the matching records from the right table. If there is no match in the right table, I still want the rows from the left table, but with NULLs in the columns of the right table."

There no such thing as "outer" join. "outer" is when you do `where b.key is null`
# Aggregations - What to use?

- When you want subtotals -> ROLLUP()
- When you want a static column -> OVER() (window functions)

# Dates

- Get part of a date as in a number -> EXTRACT( YEAR FROM 'somedate')
- Get current date time -> NOW()
- Set the a date to be the begin of X -> DATE_TRUNC('month', '25-12-2025)' = 01-12-2025; note that if you have a time as well it always gets set to 00:00:00
- Turn a timestamp into a date -> cast to date `::date`
- Add X days/years/months/etc to a date -> ` event_date + INTERVAL '10 days'`
- Create a timestamp -> `select timestamp '2012-08-01 01:00:00`
- Subtract timestamps -> `select '2012-08-31 01:00:00'::timestamp - '2012-07-30 01:00:00'::timestamp`
- Epoch - `select extract (epoch from timestamp 'xyz')` - epoch is NOT a function, is just a parameter in `extract` like `year from` or `day from`
- Generate a random series of values -> `select generate_series(timestamp '2012-10-01 00:00:00', timestamp '2012-10-31 00:00:00', interval '1 day')`
 

# Tricks
- Round to nearest 10 (meaning if above 5 up, else down) -> ROUND(x, -1)



# Indexes and B-Trees

## Explaining the output of EXPLAIN ANALYZE

```SQL
explain analyze select * from film where rating = 'PG-13';
```

outputs...

```
Seq Scan on film (cost=0.00..100.50 rows=223 width=384) (actual time=0.013..0.225 rows=223 loops=1)" " Filter: (rating = 'PG-13'::mpaa_rating)" " Rows Removed by Filter: 777" "Planning Time: 0.060 ms" "Execution Time: 0.247 ms
```

Breaking it down:

```
Seq Scan on film (cost=0.00..100.50 rows=223 width=384)
```

`Seq Scan` means the database is doing a sequential scan instead of an index. This means it's going through the entire table and not querying an index which is possibly more efficient

```
(cost=0.00..100.50 rows=223 width=384)
```

`cost = 0.00` refers to the initial cost of the query in terms of **I/O operations and CPU usage**. In this case it's zero because there's nothing to prepare.

`100.58` is the estimated cost of executing the entire query.

`rows=223` means that 223 rows will be returned as part of this query

`width=384` is the estimated size of each row in *bytes*

```
(actual time=0.013..0.225 rows=223 loops=1)
```

`actual time` is the actual time it took to execute the query

`time=0.013` means is the time in milliseconds it took to start the query and `0.225` is how long it took to complete the query and, in this case, process the 223 rows. `rows` is the number of rows returned and `loops` is the number of times the query was executed.

```
Filter: (rating = 'PG-13'::mpaa_rating)" " Rows Removed by Filter: 777"
```

This is what filter got applied and how many rows were removed by the filter.

```
Planning Time: 0.060 ms
```

How long PG took planning how to execute the query

```
Execution Time: 0.247 ms
```

How long it took to execute the query.


## EXPLAIN ANALIZE with Indexes

```
"Bitmap Heap Scan on film  (cost=5.32..99.26 rows=102 width=384) (actual time=0.048..0.120 rows=118 loops=1)"
"  Recheck Cond: ((rating = 'PG-13'::mpaa_rating) AND (length > 120))"
"  Heap Blocks: exact=48"
"  ->  Bitmap Index Scan on idx_film_rating_length  (cost=0.00..5.29 rows=102 width=0) (actual time=0.034..0.035 rows=118 loops=1)"
"        Index Cond: ((rating = 'PG-13'::mpaa_rating) AND (length > 120))"
"Planning Time: 0.322 ms"
"Execution Time: 0.165 ms"
```

The output is slightly different but the interpretation is the same. However note that to be able to compare two queries you need to add up the planning time and the execution time.