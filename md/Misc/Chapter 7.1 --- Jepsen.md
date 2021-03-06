# Jepsen
Notes on [this sequence of blog posts](https://aphyr.com/tags/jepsen) exploring data properties of various databases.

## Postgres
* Postgres has various options for consistency, up to and including 2PL serialized isolation.
* However the article focuses on the communication between the client and the server.
* Postgres uses a **two-phase commit** (Byzantine generals problems).
* In order for a commit to succeed on the client side, the (1) transaction must go through and (2) the database must respond to the client with a success.
* If a network partition between the client and the service occurs in between the write and the ack, the client will respond with a failure timeout.
* The data will still be modified, however.
* So if a network partition occurs, and a failure bubbles up, technically services relying on Postgres cannot know if the transaction succeeded or not.
* Is it hard to create a partition between the database and the client? Maybe.

## Redis
* Redis by default runs on a single server. It offers serialized isolation (e.g. the highest possible guarantee level) via actual serialization (everything runs on one thread, and transactions are sequenced on that thread).
* In this configuration Redis is CP.


* Redis can be made highly available.
* It offers asynchronous single-leader replication (see Chapter 6 notes).
* A separate service, called Sentry, is used to detect serious network partitions.
* If a network partition occurs, a quorom of nodes (at least $N / 2 + 1$, so that only one quorom may exist) assembles and elects a new leader.
* The quorom then instructs any client connections to abandon the old configuration and use the new configuration. Nodes are added back as the partition heals and the offline nodes come back online.
* Redis does not gaurantee durability. Replication and disc writes are performed asynchronously, so data that was updated on "lost" nodes may not exist in the new quorom (having a quorom protects against this, but as always it's an availability-consistency tradeoff).


* By default, during a network partition the old primary will continue to accept and deal with requests.
* This will continue until the partition heals and the new quorom-elected leader can reach the primary again. The old primary will be told to step down.
* The data that was accepted by the old primary and acked will be lost!


* Any service built on Redis must be ready to deal with failovers that demolish consistency.
* So in the single-leader replicated configuration, Redis is not consistent.
* Redis is not available, either. If there is a failover there's no node that will accept operations.
* The tradeoff is that it's very fast.
* Caches don't care about consistency loss, which is why Redis is so good for this purpose (the difficulty of cache invalidation notwithstanding).


* You can optionally configure how long until the old primary stops accepting requests. 
* This essentially requires an occassional quorom ack on the primary, which slows the system down. I don't recommend it.
* Really, if you want to not lose data and not deal with acknowledged consistency problems don't use Redis! It wasn't designed for this!


## MongoDB

* MongoDB also uses a single-leader replication with quorom recovery.
* By default MongoDB also uses a two-phase commit against the leader (apparently it used to not even check if a write succeeded on the leader!).
* If a network partition occurs, similarly to Redis the majority quorom will elect a new leader.
* In the meanwhile, the old leader will continue to accept and ack operations.
* Once the partition heals, and the old leader rejoins the pack, the intervening data written to the old leader and not to the new leader is rolled back (specifically, to a rollback file that a system administrator can look at).
* The article claims it's not possible to get split-brain, but it seems extremely possible to get split-brain...
* So by default MongoDB is not consistent.


* As with Redis you can tell MongoDB to use majority acknowledgement. This makes it properly CP, but increases latency by a lot.

## More MongoDB

* Jespen found the Redis consensus algorithm inscruitable, with a wide range of possible edge cases. Again, use it for speed, not for any gaurantees of anything.
* The MongoDB consensus algorithm is less topologically diverse.
* Jespen went very, very deep into the MongoDB algorithms...interesting case analysis follows.


* MongoDB now has two consensus protocols, v0 and v1.
* v1 is the new default.


* MongoDB is designed to reach consensus over a log of database operations called an **oplog**.
* If a network partition occurs, divergent data may be written both to the old leader and to a new quorom-elected leader. This causes divergent history, which necessitates a rollback on heal.
* Distributed logs typically maintain a **commit point**, a furthest index in the log. Ops before the commit point are safe, but ops after the commit point are not gauranteed to be retained (see e.g. GFS).
* MongoDB however applies and acks operations immediately.
* This causes dirty reads and lost updates.
* v0 cannot prevent dirty reads, but it can prevent lost updates using the majority write concern (the leader will not ack until it knows that at least half of the follower nodes have the update).


* In case of a network partition, non-majority write durability hinges on the election of the most up-to-date follower in the quorom. However how do you determine who's furthest ahead?
* In v0 each write to the log is assigned an **optime**. That optime is a combination of the sequence order of the operation, and the wall clock time on the node at the time of the log write.
* The majority will elect the node that has the highest optime.
* In theory this will always be the node which is furthest ahead. In practice, clocks can drift, so this is not always so.
* Thus if the old leader is on a fast clock, and the newly elected leader is on a slow clock, MongoDB may in practice choose to preserve the old leader's data, not the new leader's data!
* So lost updates are non-deterministic in the sense that it can occur on either side of the partition.
* Jespen makes a lot of hooplah about this.


* v1 aimed to address these problems. v1 is the new default replication protocol.
* It takes elements from the Raft consensus algorithm (TODO: look into Raft).
* In version 0 there are election IDs. These are used for determining which primary node is the most recently voted-upon primary.
* In order to prevent double-voting, nodes were equipped with a 30-second cooldown on voting, but this was very imperfect.
* In v1 election IDs are not election terms, assigned to every node in the cluster, which increment every time the node votes. Thus a node can vote only once per term, a much better system.
* Optimes are no longer ordered using sequential wall clock entries, but instead `(term, timestamp)` tuples.
* Thus even if there is wall clock drift, and during the healing process it is discovered that the older election generation is ahead of the newer election generation, logical order will still be preserved.
* The other big change is that there is now snapshot isolation.
* There is a read concern level that can be set that will hold commits away from reads until the commit is confirmed to be durable.
* This addresses the dirty reads problem that v0 majority reads suffered from.
* Interesting showing of how hard it is to design a performant data system.
* These improvements were two years in the making.
* Jespen is mollified, which gives me more confidence in using MongoDB in production, as long as you have the right flags set.


## Riak
* Riak is a close commercial implementation of the Dynamo system.
* In Riak nodes are arranged in a ring.
* Incoming write data is hashed and duplicated to multiple nodes in the ring. You can choose how many nodes must acknowledge a write before the service acks.
* Incoming reads are bounced through a configurable number of nodes, with the most recent value among those nodes being the return value. You can choose many nodes must participate before the service acks.
* If a partition occurs, the nodes split off into subrings which continue to attempt to service connection as best they can.
* Lower write duplication numbers increase write availability, as fewer nodes have to be present in the ring to accept your write. They are also faster to perform. But this comes at the cost of read availability (if a network partition occurs and no node carries the data, you're out of luck).
* The partioned nodes are in a **sloppy quorom**.
* When the partition heals the nodes perform a **hinted handoff**, moving temporarily localized data onto the nodes it was originally supposed to land on.
* Dynamo and thus Raik uses an algorithm called **vector clocks** to find, after a partition heals and data comparison operations are made, conflicting data writes from different sides of the partition.
* These conflicts may be resolved by the user application. Sometimes this is easy. For example, merging shopping carts (the original Dynamo use case). Sometimes this is hard. Worst-case, last-write-wins can be used.
* Of course last-write-wins is approximate, due to the clock drift problems explored in the previous section.
* Riak will introduce high latency onto requests that are made during a network partition. This is by design; Riak really doesn't want to drop write attempts, so it will hold onto them until the backup rings come online, which can take time.
* To use Riak effectively, you should be structuring your stored objects as **CRTDs**. CRTDs are a newfangled concept of a thing. They're immutable, commutative, and associative. Thus they can be merged without data loss!
* And don't use last-write-wins because "clocks are evil", unless it doesn't matter much or there's no other way. A merge is much better.

## Summary thus far
* Practical lessons:
  * Network errors means "I don't know", not failure.
  * Node consensus does not mean data consistency.
  * Maintaining a consistent data log in the face of primary failover is hard.
  * Wall clocks are an approximation.
  * Choose the right design for your problem space.

## Zookeeper
* ZooKeeper is a distributed consistent data store.
* As explored in the previous notebook ZK is meant for small amounts of config data that have to be fully correct.
* It uses majority quoroms and offers full consistency (serialization).
* The trade-off is that writes must wait until the full quorom acknowledges a write, and the entire dataset must fit in memory.
* Jepsen says: Zookeeper is great!


## NouDB
* This database was posed as a refutation of CAP by some famous guy.
* It is not.
* It is basically a CP database that blocks forever when a partition occurs. "Trivial availability" according to one reading of the conjecture.


## Kafka

* Kafka is a message broker, not a database. It added replication as a feature.
* Kafka replication is based on a two-tier architecture.
* Brokers that ack right up to the latest write compose an in-sync replica set. The system will continue to operate normally in the case of broker failures, right up to only having a single broker alive.
* Brokers push to data replicas, which handle actually storing the data. The data replicas are handled by Zookeeper, and are majority quorom based. Thus they can tolerate only N/2 failures.
* Brokers are more resiliant than the underlying data store.


* Jepsen showed that a complex failure in both the broker and the node layer can cause lost updates.
