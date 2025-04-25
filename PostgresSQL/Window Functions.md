
What?  Window functions as the name implies operate on a window of your data but only after the WHERE clause and all standard aggregation is applied.

If you don't specify a `PARTITION BY` or `ORDER BY` , by default the window function **looks at the entire result set.** If you specify a partition or ordering,  **it looks at all the rows up to the current row and backwards**. For example:

```C#
select staff_id, sum(staff_id) OVER(order by rental_date) as rolling_staff_id from rental 
order by rental_date
```
  
  Will output something like:
  
| staff_id | rolling_staff_id |
| -------- | ---------------- |
| 1        | 1                |
| 1        | 2                |
| 1        | 3                |
| 2        | 5                |
| 1        | 6                |
| 1        | 7                |
| 2        | 9                |
| 2        | 11               |
| 1        | 12               |
| 2        | 14               |


*Note: this example is a bit daft because rolling staff id makes no sense but you get the picture*

## PARTITION BY

Window functions have the concept of partitioning. This is hard to explain so let's do it visually. Imagine you have a database with customers and you want to add a rolling sum of how much money a certain customer as paid up to a point.

A snapshot of your table looks like this.

| ID    | CustomerID | StoreID | ProductID | Price | Timestamp                  |
| ----- | ---------- | ------- | --------- | ----- | -------------------------- |
| 17503 | 341        | 2       | 1520      | 7.99  | 2007-02-15 22:25:46.996577 |
| 17504 | 341        | 1       | 1778      | 1.99  | 2007-02-16 17:23:14.996577 |
| 17505 | 341        | 1       | 1849      | 7.99  | 2007-02-16 22:41:45.996577 |
| 17506 | 341        | 2       | 2829      | 2.99  | 2007-02-19 19:39:56.996577 |
| 17507 | 341        | 2       | 3130      | 7.99  | 2007-02-20 17:31:48.996577 |
| 17508 | 341        | 1       | 3382      | 5.99  | 2007-02-21 12:33:49.996577 |
| 17509 | 342        | 2       | 2190      | 5.99  | 2007-02-17 23:58:17.996577 |
| 17510 | 342        | 1       | 2914      | 5.99  | 2007-02-20 02:11:44.996577 |
| 17511 | 342        | 1       | 3081      | 2.99  | 2007-02-20 13:57:39.996577 |
| 17512 | 343        | 2       | 1547      | 4.99  | 2007-02-16 00:10:50.996577 |

A regular window function that calculates a rolling sum of all payments would look like this:

```SQL
select 
customer_id,
amount,
payment_date,
SUM(amount) OVER(order by payment_date) as rolling_payment
from payment
```

And would output the following:

| Customer ID | Amount | Payment Date               | Rolling Payment |
|-------------|--------|----------------------------|-----------------|
| 341         | 7.99   | 2007-02-15 22:25:46.996577 | 7.99            |
| 341         | 1.99   | 2007-02-16 17:23:14.996577 | 9.98            |
| 341         | 7.99   | 2007-02-16 22:41:45.996577 | 17.97           |
| 342         | 5.99   | 2007-02-17 23:58:17.996577 | 23.96           |
| 341         | 2.99   | 2007-02-19 19:39:56.996577 | 26.95           |
| 342         | 5.99   | 2007-02-20 02:11:44.996577 | 32.94           |
| 342         | 2.99   | 2007-02-20 13:57:39.996577 | 35.93           |
| 341         | 7.99   | 2007-02-20 17:31:48.996577 | 43.92           |
| 341         | 5.99   | 2007-02-21 12:33:49.996577 | 49.91           |
| 342         | 4.99   | 2007-03-01 00:46:38.996577 | 54.90           |

Notice that it's all mixed up. You don't have the sum for customer 341 only for example. So how would you achieve that? By adding a `PARTITION BY` clause. Like this:

```SQL
select 
customer_id,
amount,
payment_date,
SUM(amount) OVER(PARTITION BY customer_id order by payment_date) as rolling_payment
from payment
```

A `PARTITION BY`  clause is a way of saying 

> "Apply the window function only to a slice of the data rather than the whole table."

The output will now be:

| Customer ID | Amount | Payment Date               | Rolling Payment |
|-------------|--------|----------------------------|-----------------|
| 341         | 2.99   | 2007-04-08 06:40:43.996577 | 81.83           |
| 341         | 4.99   | 2007-04-09 22:30:16.996577 | 86.82           |
| 341         | 0.99   | 2007-04-10 20:32:45.996577 | 87.81           |
| 341         | 2.99   | 2007-04-27 17:34:06.996577 | 90.80           |
| 341         | 2.99   | 2007-04-29 13:56:50.996577 | 93.79           |
| 341         | 9.99   | 2007-04-29 16:41:15.996577 | 103.78          |
| 342         | 5.99   | 2007-02-17 23:58:17.996577 | 5.99            |
| 342         | 5.99   | 2007-02-20 02:11:44.996577 | 11.98           |

**Again note that because we haven't specified a window of rows ahead/preceding and we have specified a partitioning, the window function looks at all the rows up to the current row and backwards.**

## Common window functions:

- ROW_NUMBER;
- RANK
- NTILE -> divide things into buckets
- AVG 
	- Think rolling averages
	-  `select payday, daily_amount, AVG(daily_amount) OVER(order by payday ROWS BETWEEN 6 PRECEDING AND CURRENT ROW ) from daily_sales`
- LAG
	-  `LAG(expression, offset, default_value) OVER (PARTITION BY column ORDER BY column)`
	- Look at the previous X rows
- LEAD
	- Same as LAG but looking ahead
	