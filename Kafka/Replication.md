# What is Kafka replication?

In Apache Kafka, **replication** is the mechanism that ensures high availability, fault tolerance, and data durability. Because Kafka is a distributed system, it anticipates that hardware will fail, nodes will crash, and networks will hiccup. Replication ensures that even if a server dies, your data isn't lost and your applications keep running.

Here is a breakdown of how it works under the hood.

## The Core Concept: Topics, Partitions, and Replicas

To understand replication, you have to look at how Kafka structures its data:

- **Topics** are divided into **Partitions** (the fundamental unit of scalability).
- When you configure replication, you set a **Replication Factor** (e.g., a replication factor of 3 is the industry standard).
- This means every single partition will have **3 identical copies (replicas)** distributed across different brokers (servers) in the cluster.

## Leaders vs. Followers

Not all replicas are created equal. For every partition, Kafka assigns roles to the replicas:

### 1. The Leader Replica

Each partition has exactly **one** Leader.

- By default, all producer writes and consumer reads go directly to the Leader.
- The Leader is responsible for receiving the data, assigning it an offset, and writing it to its local log.

### 2. The Follower Replicas

The remaining copies are Followers.

- Followers do not typically serve client requests (though newer Kafka versions allow reading from closest followers in specific setups).
- Their sole job is to constantly pull data from the Leader to stay up-to-date, acting like passive clones.

## In-Sync Replicas (ISR)

How do we know if a Follower is actually doing its job? Kafka tracks this using the **ISR (In-Sync Replicas)** list.

- **ISR** is the subset of replicas that are actively keeping up with the Leader.
- If a Follower crashes or falls too far behind (defined by the configuration `replica.lag.time.max.ms`), it is kicked out of the ISR list.
- If the Leader dies, Kafka will **only** elect a new Leader from the pool of In-Sync Replicas to guarantee no data loss.

## How Writing Data Works (The `acks` Setting)

When a producer writes data to Kafka, replication behavior is dictated by the `acks` (acknowledgments) configuration:

- **`acks=0`**: The producer doesn't wait for a response. High speed, high risk of data loss.
- **`acks=1`**: The producer waits for the **Leader** to write the data to its local log. Once the leader acknowledges, the producer moves on. (Risk: If the leader dies before followers copy it, data is lost).
- **`acks=all` (or `-1`)**: The producer waits until the Leader **and all active ISRs** have successfully written the data. This provides the highest durability.

## Summary of Benefits

| Feature | Description |
| --- | --- |
| **Fault Tolerance** | If a broker hosting a Leader partition crashes, another broker holding an ISR copy instantly takes over. |
| **No Data Loss** | With `acks=all` and a replication factor of 3, you can lose up to 2 brokers simultaneously without losing a single message. |
| **Zero Downtime** | Cluster rebalancing and leader elections happen automatically in milliseconds, completely transparent to your applications. |

