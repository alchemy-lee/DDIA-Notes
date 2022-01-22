# Chapter 5 Replication

*Replication* means keeping a copy of the same data on multiple machines that are connected via a network. There are several reasons why you might want to replicate data:

- To keep data geographically close to your users (reduce latency)
- To allow the system to continue working even if some of its parts have failed (increase availability)
- To scale out the number of machines that can serve read queries (increase read throughput)

All of the difficulty in replication lies in handling changes to replicated data. We will discuss three popular algorithms for replicating changes between nodes: *single-leader*, *multi-leader*, and *leaderless* replication.

## 5.1 Leaders and Followers

Every write to the database needs to be processed by every replica; otherwise, the replicas would no longer contain the same data. The most common solution for this is called *leader-based replication*.It works as follows:

<img src="images/image-20220122210521993.png" alt="image-20220122210521993" style="zoom:50%;" />

1. One of the replicas is designated（指定） the *leader*. When clients want to write to the database, they must send their requests to the leader, which first writes the new data to its local storage.
2. The other replicas are known as *followers*. Whenever the leader writes new data to its local storage, it also sends the data change to all of its followers as part of a *replication log*. Each follower takes the log from the leader and updates its local copy of the database accordingly, by applying all writes **in the same order** as they were processed on the leader.
3. When a client wants to read from the database, it can query either the leader or any of the followers. However, writes are only accepted on the leader.

### 5.1.1 Synchronous Versus Asynchromous Replication

An important detail of a replicated system is whether the replication happens *synchronous* or *asynchronous*.

<img src="images/image-20220122210903172.png" alt="image-20220122210903172" style="zoom:50%;" />

In the example of Figure 5-2, the replication to follower 1 is *synchronous*: the leader waits until follower 1 has confirmed that it received the write before reporting success to the user, and before making the write visible to other clients. The replication to follower 2 is *asynchronous*: the leader sends the message, but doesn’t wait for a response from the follower. There is a delay before follower 2 processes the message, but there is no guarantee of how long it might take.

- The **advantage** of synchronous replication: The follower is guaranteed to have an up-to-date copy of the data that is consistent with the leader. 

- The **disadvantage** of synchronous replication: If the synhronous follower doesn't respond, the write cannot be processed. The leader must block all writes and wait.

In practice, if you enable synchronous replication on a database, it usually means that **one** of the followers is synchronous, and the others are asynchronous. If the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous. This configuration is sometimes also called *semi-synchronous*（半同步）.

Often, leader-based replication is configured to be completely asynchronous. In this case, if the leader fails and is not recoverable, any writes that have not yet been replicated to followers are lost. However, a fully asynchronous configura‐ tion has the advantage that the leader can continue processing writes, even if all of its followers have fallen behind.

### 5.1.2 Setting Up New Followers

From time to time, you need to set up new followers—perhaps to increase the number of replicas, or to replace failed nodes. The process looks like this:

1. Take a consistent snapshot of the leader's database at some point in time.
2. Copy the snapshot to the new follower node.
3. The follower connects to the leader and requests all the data changes that have happend since the snapshot was taken.
4. When the follower has processed the backlog of data changes since the snapshot, we say it has *caught up*. It can now continue to process data changes from the leader as they happen.





## 5.3 Multi-Leader Replication

Leader-based replication has one major downside: there is only one leader, and all writes must go through it. If you can’t connect to the leader for any reason, you can’t write to the database. 

Multi-leader configuration: allow more than one node to accept writes, each node that pro‐ cesses a write must forward that data change to all the other nodes. In this setup, each leader simultaneously acts as a follower to the other leaders.

### 5.3.1 Use Cases for Multi-Leader Replication

