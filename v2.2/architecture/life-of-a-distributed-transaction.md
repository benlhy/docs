---
title: Life of a Distributed Transaction
summary: This guide details the path through CockroachDB's architecture that a query takes, starting with a SQL client and progressing all the way to RocksDB (and then back out again).
toc: true
---

Because CockroachDB is a distributed transactional database, the path queries take is dramatically different from many other database architectures. To help familiarize you with CockroachDB's internals, this guide covers what that path actually is.

If you've already read the [CockroachDB architecture documentation](https://www.cockroachlabs.com/docs/stable/architecture/overview.html), this guide serves as another way to conceptualize how the database works. This time, instead of focusing on the layer's of CockroachDB's architecture, we're going to focus on the linear path that a query takes through the system (and then back out again).

To get the most out of this guide, we recommend first starting with the architecture documentation, starting with the [overview](https://www.cockroachlabs.com/docs/stable/architecture/overview.html) and progressing through all of the following sections. This guide provides brief descriptions of each component's function and links where appropriate, but assumes the reader has a basic understanding of the architecture in the first place.

## Overview

This guide is organized by the physical actors in the system, and then broken down into the components of that actor in the sequence in which they're involved.

Here's a brief overview of the physical actors, in the sequence with which they're involved in executing a query:

1. **SQL Client** sends a query to your cluster.
1. **Load Balancing** routes the request to your cluster.
1. **Gateway** is a CockroachDB node that processes the request and responds to the client.
1. **Leaseholder** is a CockroachDB node responsible for serving reads and coordinating writes of a specific range of keys in your query.
1. **Raft leader** is a CockroachDB node responsible for maintaining consensus among your CockroachDB replicas.

Queries roughly traverse these actors in reverse order once the transaction completes. We say "roughly" because there might be many leaseholders and Raft leaders involved in a single query, and there is little-to-no interaction with the load balancer during the response.

## SQL Client/Postgres Wire Protocol

To begin, a SQL client (e.g., an app) performs some kind business logic against your CockroachDB cluster, such as inserting a new customer record.

This request is sent over a connection to your CockroachDB cluster that's established using a PostgreSQL driver.

## Load Balancing & Routing

Modern architectures require distributing your cluster across machines to improve throughput, latency, and uptime. This means queries are routed through load balancers, which choose the best CockroachDB node to connect to. Because all CockroachDB nodes have perfectly symmetrical access to data, this means your load balancer can connect your client to any node in the cluster and access any data while still guaranteeing strong consistency.

Your architecture might also have additional layers of routing to enforce regulatory compliance, such as ensuring GDPR compliance.

Once your router and load balancer determine the best node, your client's connection is established to the gateway node.

## Gateway

The gateway node handles the connection with the client, both receiving and responding to the request.

### Parsing

The gateway node first [parses](sql-layer.html#sql-parser-planner-executor) the client's SQL statement to ensure it's valid according to the CockroachDB dialect of PostgreSQL.

While CockroachDB presents a SQL interface to clients, the actual database is built on top of a key-value store. Parsed SQL statements are converted into key-value operations, and a the node [generates a plan](sql-layer.html#logical-planning) to execute the query.

### TxnCoordSender

The newly generated plan is sent to the `TxnCoordSender`, which performs a large amount of the accounting and tracking for a transaction, including:

- Accounts for all keys involved in a transaction. This is used, among other ways, to manage the transaction's state.
- Packages all key-value operations into a `BatchRequest`, which are forwarded on to the node's `DistSender`.

### DistSender

The gateway node's `DistSender` receives `BatchRequests` from the `TxnCoordSender`. It dismantles the initial `BatchRequest` by taking each operation and finding which physical machine should receive the request for the range––known as the range's leaseholder. The address of the range's current leaseholder is readily available in both local caches, as well as in the [cluster's `meta` ranges](distribution-layer.html#meta-range-kv-structure).

These dismantled `BatchRequests` are reassembled into new `BatchRequests` containing the address of the range's leaseholder.

All writes operations also propagate the leaseholder's address back to the `TxnCoordSender`, so it can track and clean up write operations as necessary.

The `DistSender` sends out the first `BatchRequest` for each range in parallel. As soon as it receives a provisional acknowledgment from the leasholder node’s evaluator (details below), it sends out the next `BatchRequest` for that range.

The DistSender then waits to receive commit acknowledgments for all of its write operations, as well as values for all of its read operations.

<!-- #### Finding leaseholders (meta ranges)

Each range is replicated at least three times, and one of these range is chosen as the leaseholder, which serves and reads and coordinates all writes for the range. Because of this architecture, all of the KV operations the `DistSender` receives must be routed directly to the range's leaseholder.

To determine which physical node to send the operation to, the `DistSender` relies on data from the cluster's meta ranges, which detail which physical nodes contain the range's replicas as well as which node is the leaseholder. For any given request, the `DistSender` can determine the leaseholders by performing a read from the meta ranges for the keys in question.

For example, if the `DistSender` needs to route a request for keys `[1000, 2000)`, it performs a read across the meta ranges for those values. The response contains the leaseholder as the first node in the response.

To optimize performance, nodes heavily cache the values they've previously seen in meta ranges, which means that in most cases this read is served by a local cache, rather than going all the way out to the network. -->

<!-- #### Transaction Record

The first range that a transaction touches receives an additional KV operation to create a translation record (`TxnRecord`). This key-value pair is used to store the transaction's current state for the life of the transaction. It starts off in a `PENDING` state, and then progresses to either `COMMITTED` if the transaction succeeds or `ABORTED` if the transaction must be canceled.  -->

## Leaseholder node

The gateway node's `DistSender` tries to send its `BatchRequests` to the replica identified as the range's [leaseholder](replication-layer.html#leases), which is a single replica that serves all reads for a range, as well as coordinates all writes. Leaseholders play a crucial role in CockroachDB's architecture, so it's a good topic to make sure you're familiar with.

### Request response

Because the leaseholder replica can shift between nodes, all nodes must be able to return a request for any key.

##### No Longer Leaseholder

If a node:

- *Was* the leaseholder, but no longer holds the lease
- Holds a replica for the range

Then the node denies the request, but includes the last known address for the leaseholder of that range.

Upon receipt of this response, the `DistSender` will update the header of the `BatchRequest` with the new address, and then resend the `BatchRequest` to the newly identified leaseholder.

##### No Longer Has/Never Had Range

If a node doesn't have a replica for the requested range, it denies the request without providing any further information.

In this case, the `DistSender` must look up the current leaseholder using the [cluster's `meta` ranges](distribution-layer.html#meta-range-kv-structure) range.

##### Success

Once the node that contains the leaseholder of the range receives the `BatchRequest`, it begins processing it, and progresses onto checking the timestamp cache.

### Timestamp cache

The timestamp cache tracks the highest timestamp (i.e., most recent) for any read operation that a given range has served.

Each write operation in a `BatchRequest` checks its own timestamp versus the timestamp cache to ensure that the write operation has a higher timestamp; this guarantees that history is never rewritten and you can trust that reads always served the most recent data. If a write operation fails this check, it must be restarted at a timestamp higher than the timestamp cache's value.

### Latch manager

Operations in the `BatchRequest` are serialized through the leaseholder's latch manager.

This works by giving each write operation a latch on a row. Any reads or writes that come in after the latch has been granted on the row must wait for the write to complete, at which point the latch is released and the subsequent operations can continue.

### Evaluator

The evaluator ensures that write operations are valid, e.g., they pass any `CHECK` constraints on the data. This is trivial in many cases because the evaluator can simply check the leaseholder's data, which is guaranteed to be valid and up-to-date.

If the write operation is valid according to the evaluator, the leaseholder sends a provisional acknowledgment to the gateway node's `DistSender`; this lets the `DistSender` begin to send its subsequent `BatchRequests` for this range.

Importantly, this feature is entirely built for transactional optimization (known as [transaction pipelining](transaction-layer.html#transaction-pipelining)). There are no issues if an operation passes the evaluator but doesn't end up committing.

### Writing the transaction record

The very first range that a transaction interacts with also receives a write for a key known as its transaction record. This includes a UUID to uniquely identify the transaction, as well as the canonical record of its state––e.g. `PENDING`, `ABORTED`, `COMMITTED`. This record is used by all operation to determine the state of the transaction. For example, when a write is blocked by another write, it periodically checks its own transaction record to ensure that it hasn't been aborted.

As soon as the write for the transaction record completes (i.e., achieves consensus through Raft), it begins communicating with the gateway node's `TxnCoordSender`, which regularly heartbeats the transaction to keep it alive (i.e., in a `PENDING` state). If the heartbeating from the TxnCoordSender stops, the TxnRecord moves itself into `ABORTED` status.

This write is handled like all other writes, which we'll discuss in more detail in a subsequent section.

### Reads from RocksDB

All operations (including writes) begin by reading from the local instance of RocksDB to check for write intents for the operation's key. We talk much more about [write intents in the transaction layer of the CockroachDB architecture](transaction-layer.html#write-intents), which is worth reading, but a simplified explanation is that these are provisional, uncommitted writes that express that some other concurrent transaction plans to write a value to the key.

What we detail below is a simplified version of the CockroachDB transaction model. For more detail, check out [the transaction architecture documentation](transaction-layer.html).

#### Resolving Write Intents

If an operation encounters a write intent for a key, it attempts to "resolve" the write intent by checking the write intent's transaction's state. If the write's intent's transaction is...

- `COMMITTED`, this operation converts the write intent to a regular key-value pair, and then proceeds as if it had read that value instead of a write intent.
- `ABORTED`, this operation discards the write intent and reads the next-most-recent value from RocksDB.
- `PENDING`, the new transaction attempts to "push" the write intent's transaction by moving that transaction's timestamp forward (i.e. ahead of this transaction's timestmap). The push succeeds only if this transaction's priority is explicitly higher; i.e. it has a `HIGH` priority or the transaction whose write intent it encountered has a `LOW` priority.
  
  If the push succeeds, the operation continues.

  If this push fails, this transaction goes into the [`TxnWaitQueue`](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#txnwaitqueue) on this node. Only once the blocking transaction completes (i.e., commits or aborts), the incoming transaction can continue.

Check out our architecture documentation for more information about [CockroachDB's transactional model]((https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html). 

#### Read Operations

If the read doesn't encounter a write intent and the key-value operation is meant to serve a read, it can simply use the value it read from the leaseholder's instance of RocksDB. This works because the leaseholder had to be part of the Raft consensus group for any writes to complete, meaning it must have the most up-to-date version of the range's data.

The leaseholder aggregates all read responses into a `BatchResponse` that will get returned to the gateway node's `DistSender`.

As we mentioned before, each read operation also updates the timestamp cache.

### Write Operations

After guaranteeing that there are no existing write intents for the keys, `BatchRequest`'s key-value operations are converted to [Raft operations](replication-layer.html#raft) and have their values converted into write intents.

The leaseholder then proposes these Raft operations to the Raft group leader. Most often, the leaseholder and the Raft leader are the same node, but there are situations where the roles might drift to different nodes. However, when the two roles are not collocated on the same physical machine, CockroachDB will attempt to relocate them on the same node at the next opportunity.

## Raft Leader

CockroachDB leverages Raft as its consensus protocol. If you aren't familiar with it, we recommend checking out the details about [how CockroachDB leverages Raft](https://www.cockroachlabs.com/docs/v2.1/architecture/replication-layer.html#raft), as well as [learn more about how the protocol works at large](http://thesecretlivesofdata.com/raft/).

In terms of executing transactions, the Raft leader receives proposed Raft commands from the leaseholder. Each Raft command is a write that is used to represent an atomic state change of the underlying key-value pairs stored in RocksDB.

### Consensus

For each command the Raft leader receives, it proposes a vote to the other members of the Raft group.

Once it achieves consensus (i.e. a majority of nodes including itself acknowledge the Raft command), it is committed to the Raft leader’s Raft log, and written to RocksDB. The other nodes move their Raft log ahead as soon the Raft leader indicates the group reached consensus on the write.

Once the leader commits the Raft log entry, it’s considered committed. At this point the value is considered written, and if another operation comes in and performs a read on RocksDB for this key, they’ll encounter this value.

Note that this write operation creates a write intent; these writes will not be fully committed until the gateway node's `TxnCoordSender` sets the transaction record's status to `COMMITTED`.

## On the way back up

Now that we have followed an operation all the way down from the SQL client to RocksDB, we can pretty quickly cover what happens on the way back up (i.e., generating a response to the client).

1. Once the leaseholder applies a write to its Raft log, it sends an commit acknowledgment to the gateway node's `DistSender`, which was waiting for this signal (having already received the provisional acknowledgment from the leaseholder's evaluator).
1. The gateway node's `DistSender` aggregates commit acknowledgments from all of the write operations in the `BatchRequest`, as well as any values from read operations that should be returned to the client.
1. Once all operations have successfully completed, the `DistSender` sends a response to the `TxnCoordSender`, indicating that the transaction completed.
1. The `TxnCoordSender` checks the transaction record. There are a few situations that might arise here:
	- If the transaction's timestamp was moved, it performs a [read refresh](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#read-refreshing) to see if any values it needed have been changed. If the read refresh is successful, the transaction can commit at the pushed timestamp. If the read refresh fails, the transaction must be restarted.
	- If the transaction is in an `ABORTED` state, the `TxnCoordSender` notifies the SQL interface.
	- If the transaction is still `PEDNING`, it's moved to `COMMITTED`. Note that this modification of the transaction record is handled as an update to the key-value pair, meaning it must go through Raft like all other write operations. Once the transaction record is moved to `COMMITTED`, the transaction is considered committed.
1. The `TxnCoordSender` responds to the SQL interface with the value that should be returned to the client (e.g. the number of affected rows)
  The `TxnCoordSender` also begins asynchronous intent cleanup by sending a request to the `DistSender` to convert all write intents it created for the transaction to fully committed values. However, this process is largely an optimization; if any operation encounters a write intent, it checks the write intent's transaction record. If the transaction record is `COMMITTED`, the operation can perform the same cleanup and convert the write intent to a fully committed value.
1. The SQL interface then responds to the client, and is now prepared to continue accepting new connections.