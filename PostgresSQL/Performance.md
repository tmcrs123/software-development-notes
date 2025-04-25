
# Bloat

Postgres uses something called a MVCC model. This stands for "Multi Version Concurrency Control".

What does this mean?

This relates to the way PG works when updating and deleting rows.

Whenever PG updates or deletes a row it does NOT update the row that you are targeting, it creates a new row version (copy?) for that update and marks the old row as "dead".

The old row is only disposed of when a process called "Vaccum" occurs (more on that later).

Your database may become bloated if you update and delete a lot before allowing time for the vaccum procress to run.

Bloat can occur in tables and indexes(how?)