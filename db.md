# Consistency

## SQL Replication Modes

- asynchronous

Replication in MySQL/PostgreSQL is asynchronous by default - the primary does not wait for a replica to indicate that it wrote the data. 

- synchronous

With synchronous replication, the primary will wait for any or all replicas to confirm that they received and wrote the data. 

- semi-synchronous (MySQL)

## Relational DB strong consistency

Relational DB replication does not have immediate data consistency but is still considered as a strong consistency.

A replication replica is always lagging behind at least a fraction of a second, because the master doesn't write changes to its binary log until the transaction commits, then the replica has to download the binary log and relay the event.

But the changes are still atomic. You can't read data that is _partially changed_. You either read committed changes, in which case all constraints are satisfied, or else the changes haven't been committed yet, in which case you see the state of data from before the transaction began.

So you might temporarily read old data in a replication system that lags, but you won't read _inconsistent data_.

Whereas in an "eventually consistent" system, you might read data that is partially updated, where the one account has been debited but the second account has not yet been credited. So you can see inconsistent data. 
