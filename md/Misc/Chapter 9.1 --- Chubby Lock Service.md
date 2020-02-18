# Chapter 9.1 --- Chubby Lock Service

* A **distributed lock service** is a service which provides distributed nodes a common lock implementation.
* These locks are used for access to shared resources in a replication environment.
* Chubby is Google's implementation of a distributed lock service, and it was very influential in further designs.
* A distributed lock service can be used as a simple name server, and Chubby is often deployed for this use-case as well at Google (the alternative is using something like DNS).
* In particular, it's often used to achieve distributed master elections (as in e.g. choosing a GFS master).


* Chubby is a working Paxos wrapper, distributed using the lock service abstraction because that is what is most familiar to sequential-oriented developers.
* Its use cases are all course-grained lock use cases. Thus the design prioritizes consesus over throughput.


* A Chubby cluster has some number of machines (five is typical), of which one is the master and the rest are follower nodes.
* Followers only copy the data. The master handles all of the reads and writes.
* A master lease is given to the master, with a certain timeout. Nodes that vote on the master promise not to vote again until the cooldown is over, so an elected master is assured that it is the true master during that interval, and can act without consulting followers.
* NB: this timeout pattern is what caught MongoDB out in the field. An epoch pattern, like in MongoDB v1, is probably better.
* The resources being locked are files. The file system used is UNIX-like, but simplified.
* Chubby locks have two states: exclusive, for writing, and shared, for reading.


* An action on a distributed file is dependent on prior observed file state. If the state changes in the middle of a read-modify-write loop, for example, the data written may no longer be valid.
* Actions against chubby locked resources must provide isolation gauarantees (this is a distributed service!) to be useful, otherwise corruption can occur due to unsafe writes.
* Recall that providing full isolation requires strong sequential ordering.
* The Chubby solution is to have clients ask, separately, for sequence numbers. When the client finally performs the action, it provides Chubby with this number. Chubby checks the sequence string to determine whether or not the action is still valid (presumptions are still satisfied).


* Chubby also implements a cache layer on top.


* In the case of a failover, there is a grace period after a lock lease ends during which the lock holder continues to hold the lock, and blocks the application layer, but doesn't perform any actions (so as to avoid inconsistent data).
* If a master election succeeds before the grace period ends, the lock client and the master perform a three-roundtrip lease reaquisition dance.
* If the master election does not succeed before the grace period ends, the lock client gives up and reports an error to the application layer.
* Essentially configurable availability timeout behavior.


* After a failover the master has to replicate the state of the old master in a state-preserving way. This means:
  * Lock-giving and taking-away operations are synchronously replicated to disc (via consensus algorithms, specifically Paxos).
  * Lock lease extensions are not synchronously replicated to disc. Instead, when a failover occurs, the new master refreshes the lease to the maximum lease the old master could possibly have given.
    * Tradeoff between locks being held too long in case of a failover and lease extension speed.
    * Synchronous replication is slow, asynchronous replication doesn't provide the necessary strong consistency guarantee.
    * Lock hold length is un-important in a course-grained locking system. 
    * Lease extensions happen often, by contrast.
    * Speed over failover holds is worth it.
    
    
* How is Chubby used as a name service, and why is that advantageous?
* The primary name service competitor is DNS.
* DNS records have a time-to-live (TTL), which manages how long they live in cache.
* It's impossible to flush the DNS cache globally (?), you have to just wait until the TTLs expire.
* To provide rapid name changes (so that services aren't blocked by 404s to endpoints that have moved), you have to set a small TTL. But a small TTL results in more Keep-Alive traffic, which can overwhelm the DNS server.
* A Chubby-based name service recipe allows an application to finely control endpoint changes.
* To populate a service, take a lock on a file describing the endpoint.
* To move a service, take a write lock and change the endpoint description.
* Instant change propagation!
