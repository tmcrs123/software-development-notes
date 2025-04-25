Imagine you have a query like this:


```sql
select c.customer_id, c.first_name || ' ' || c.last_name as full_name,
		  --total amount spend
		 (
			 select sum(amount) from payment p
			 where p.customer_id = c.customer_id
		 ) as total_spent
```

This is called a correlated subquery.

It's correlated because of the `where` clause in the inner query.

On the surface this might look like a `INNER JOIN` but this is **NOT** a join. This is a filtering operation.

In this case, under the hood ==we are looping for every row in the customer table== and filtering where the `customer_id` from the row matches the `customer_id` on the payments table. Unlike a `JOIN` this does not "glue" tables together.

