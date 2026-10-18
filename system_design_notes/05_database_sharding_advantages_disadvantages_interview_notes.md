# Database Sharding — Advantages, Disadvantages & Interview Notes

> Quick revision | System Design / SRE interview

## What is database sharding?

**Layman explanation:** Instead of keeping all users in one giant database, split the users across multiple databases. Each database owns a **different subset** of the data (a *shard*).

Example: An application with 10 million users might spread them across three shards.

```text
              Client
                |
           Application API
                |
         Shard routing logic
           /     |      \
          /      |       \
       Shard 1  Shard 2  Shard 3
       3M users 3M users 4M users
```

The numbers are illustrative: actual distribution and traffic may be uneven.

## Advantages

| Advantage | Easy explanation |
|---|---|
| **Horizontal scalability** | Add database capacity by spreading data across more machines. |
| **Potentially faster queries** | Queries targeting a single shard scan or work with a smaller dataset. Query/index design still matters. |
| **Load distribution** | Different shards can handle different users' reads/writes in parallel. |
| **Greater storage capacity** | Data is no longer limited to the disk of a single database server. |
| **Fault isolation** | Failure of one shard may leave unrelated shards usable, provided there are no shared critical dependencies. |

## Disadvantages

| Disadvantage | Easy explanation |
|---|---|
| **Routing complexity** | The application, middleware or database must decide where each record lives. |
| **Cross-shard joins and queries** | Finding or joining data from several shards takes coordination and may be slow. |
| **Rebalancing/data movement** | Scaling or changing the sharding scheme can require copying records between servers. |
| **Hotspots / uneven traffic** | One shard may contain much busier users or tenants than another. |
| **Cross-shard transactions** | Maintaining atomicity and consistency across shards is harder. |
| **Backup, restore and operations** | Consistent snapshots, monitoring and recovery across shards need coordination. |

## How sharding works in the backend

**Example: hash-based sharding**

```text
Alice requests her profile (user_id = 101)
               |
          Application API
               |
         Shard routing logic
      hash(user_id) % 3 = 2
               |
          Database Shard 2
               |
          Read Alice's row
               |
          Return API response
```

- The **shard key** here is `user_id`. Choose a key that distributes data/traffic well and matches common queries.
- A **shard router** can live in application code, middleware or a database system.
- `hash(user_id) % N` is simple, but **changing N remaps many keys**; production setups often use consistent hashing, fixed logical buckets or a shard-location directory to limit disruptive migrations.
- A query covering all users may need a **scatter-gather** read from multiple shards.

## Interview scenarios

**Q1. What if Shard 2 fails?**  
Users whose data is on Shard 2 may experience failures. Other shards can continue if isolated. **Replication and failover for each shard** are needed for high availability; sharding alone does not protect against data loss.

**Q2. What if Shard 3 becomes overloaded?**  
Check traffic distribution, slow queries and heavy tenants. Optimize indexes/queries, consider caching/read replicas, then split or rebalance the hot shard if necessary.

**Q3. Is sharding the same as replication?**  
No. **Sharding splits** different data across databases for capacity and scale; **replication copies** the same data to other nodes for redundancy and/or read scaling. A robust system may use both.

**Q4. When should you shard?**  
When a single database cannot meet storage or write-throughput needs despite sensible indexing, query optimization, resource scaling, caching and (for reads) replicas. Sharding adds operational cost.

## 30-second interview answer

> Database sharding divides a large dataset into smaller partitions stored on separate database servers. It increases storage and throughput capacity, distributes load, and can isolate failures. The trade-offs are complex routing, cross-shard queries and transactions, uneven load, data rebalancing, and more complicated backup/recovery. I would first optimize the existing database and use caching or read replicas where appropriate. If we shard, I would select a good shard key and replicate each shard for high availability.
