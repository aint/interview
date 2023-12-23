# CAP

- **Consistency**: all the replicas that have a particular record must return the exact same value. 
- **Availability**: all the active nodes at any moment must be able to respond to different operations.
- **Partition-tolerance**: the system must be able to tolerate network partition among its participant nodes.

![image](https://github.com/aint/interview/assets/3179252/1a9fa58c-024e-45e5-8767-f53611a8beb6)


# BASE

- **Basically Available**: a distributed system should be available to respond with some acknowledgment — even if it’s a failure message, to any incoming request.
- **Soft State**: the system may keep changing states as and when it receives new information.
- **Eventually Consistent**: the components in the system may not reflect the same value/state of a record at a given point in time. They will settle it with time, eventually, though.


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

## Consistency in ACID and CAP

Consistency in ACID is not same as Consistency in CAP. In CAP, consistency refers to the consistency of the values in different copies of the same data item in a replicated distributed system environment. In ACID, it refers to the fact that a transaction will not violate the integrity constraints specified on the database schema.
