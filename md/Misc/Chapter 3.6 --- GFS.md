# Google File System

* Google File System (GFS) is a distributed file system abstractions that is used at Google.
* It is another Jeffrey Dean invention, dating from around 2003 or so.
* It was replaced in production by another distributed file system, Collusus, is 2010.
* Still the paper on this subject is considered very influential.
* Some argue that GFS is the true innovation in big-data processing in the early-mid 2000s, not Hadoop...


* GFS was built with the assumption of constant failures in a large number of commodity parts.
* It assumes huge (in 2003 this meant multi-gigabyte) files.
* It assumes mostly append and sequential scan based workloads, not random writes, against those huge files. This emphasizes I/O speed (reliably sustained I/O moreso than latency) and deemphasizes caching.
* It needs to have native support for concurrent reads (and writes?).


* There is one master process and and a large number of worker processes. The latter are called chunkservers.
* Data is split in chunks of a certain size, and each chunk is assigned a globally unique ID.
* The master sends the chunks out to the chunkservers for storage. Usually three copies per chunk are stored, to account for possible hardware failures.
* The master polls chunkservers for health checks.
* The master maintains all top-level processes, like chunk migration, metadata storage, chunk mapping, garbage collection, and so on.


* GFS does not provide a POSIX API. You are expected to interact with it via a client.
* At read time the client asks the master for the locations of the chunks that constitute the file. It then goes and asks the chunkservers for the chunks, using the ID list the master created.
* Since the chunks are on a network of machines you can get multiple concurrent I/O, and the file-reading operation is naively only as slow as the slowest chunk read!


* GFS originally used a 64 MB chunk size, huge by OS standards.
* Each chunk is stored as a Linux file.
* This leads to file system fragmentation. To deal with it, GFS uses **lazy space allocation**. Basically, chunks smaller than 64 MB are sometimes kept in memory. Chunks are only written once they hit 64 MB in size. This minimizes fragmentation as much as possible.
* Larger chunk sizes reduce load on the master node, which is an important design consideration.
* However, this overall design can lead to hotspots, when there are chunkservers hosting many small files.


* The master stores three major types of metadata: the file and chunk namespaces, the mapping from files to chunks, and the locations of each chunk’s replicas.
* For fast access this metadata is kept in-memory. The first two are made persistent, in the event of the master crashing, by writing to a local log file (which is itself replicated elsewhere). E.g. what Apache Zookeeper and other durable metadata stores do.
* The master does not store chunk location information persistently. Instead, it asks each chunkserver about its chunks at master startup and whenever a chunkserver joins or leaves the cluster.
* This last thing was done to make keeping up with cluster changes easier.


* File creation is atomic, and is handled exclusively by the master.
* File writes and appends are treated under a concurrent write model. Appends occur at an offset of GFS's choosing, which may be past the "true last byte" of the file. Duplicate records and padding in between concurrent write regions may be inserted by GFS.
* In a multi-mutation environment (e.g. concurrent writes and appends) GFS guarantees that the last mutation goes through correctly (so if multiple writes change the same chunk, at worst the last chunk write will be reflected).
* GFS achieves this by applying mutations to a chunk in the same order on all its replicas and by using chunk version numbers to detect stale replicas that have missed mutations due to being down, and garbage collecting those.


* This is weak consistency, because it is not guaranteed that all writes succeed!
* The trade-off is concurrent writes, and hence higher possible sustained I/O volume.
* Dealing with weak consistency because the problem of application code.
* What you can do in the face of such an unreliable filesystem:
  * Don't mutate, append.
  * If you are a writer process, buffer data in-memory (or perhaps on disc) and keep it buffered until absolutely sure the write succeeded.
  * If you are a reader process, establish checkpoints and treat data from the checkpoints instead of reading from the disc.


* How is this achieved?
* When a mutation on a chunk appears the master hands a lease to one of workers handling that chunk.
* The lease lasts 60 seconds. For all ensuing in-flight mutations, that worker designates the order of operation for the writes and applies them.
* The backup workers follow suit.
* The lease may expire or, if mutations are still in flight, it may be renewed. Leases can be renewed indefinitely.
* This communication design ensures that write operations follow a total order that is the same on every node, whilst handing off as much work as possible away from the master and to the chunk node.

* Interestingly, control flow is decoupled from data flow.
* In terms of data flow, every chunkserver that having the same chunk is arranged in a linear chain on the network, regardless of which one is the current lease owner (leader).
* Mutation data is streamed down this pipe.
* The client connects to the master, is given a leader, and then connects to the leader. The leader determines write order and sends that control signal to the follower nodes.
* If the data signal arrives before the corresponding control signal, the data is just held in place until the control signal comes through. Similarly, if the control signal comes before the data signal, the control is readied for eventual arrival.


* GFS defines two special non-POSIX operations, atomic append and snapshot.



* Atomic append is an append that is guaranteed to succeed at least once. It occurs at a GFS-determined offset, however.
* Atomic append checks if the write would cause the chunk to exceed the maximum size. If so, it pads the chunk to the maximum size (to avoid fragmentation on-disc) and tells the client to write to a new chunk. If not, the leader performs the append, signals to the followers to do the same, and returns a success to the client.
* To minimize worst-case file padding behavior GFS will only allow atomic appends at 1/4 of the chunk size.
* A failed record append on a replica will cause a retry. As a result, replicas of the same chunk may contain different and partially or fully duplicated records.
* In other words GFS only guarantees at least one successful write, it does not guarantee that data is located at the same byte offset in all three files.
* GFS handles these so-called inconsistent regions in a separate process.


* Notes on snapshot go here.
* The snapshot operations makes a fast copy of a file or directory tree. It is designed to have minimal impact on write operations.
* Snapshots are essentially an implementation of **copy-on-write** in a distributed manner.
* When it recieves a snapshot request for a chunk, the master first revokes the lease for it, or if it cannot communicate with the leader node it waits for the lease to naturally expire.
* This prevents write operations during the snapshot occurring without the master's knowledge thereof, as any new writes will require a new lease.
* The master proceeds as normal until a lease request for a snapshotted chunk comes in.
* At that point the master designates a leader chunkserver and tells it to replicate that chunk locally (locally to avoid an unnecessary network move).
* Once local replication has succeeded, the master gives the client application a reference to the *replicated* chunk, instead of the original chunk. It also updates the followers to point at the new copy. The original node is now a snapshot!


* Some concerns around rackwise replica placement and the priorization of replication in the face of node loss, and how the system handles durability, follow.


* What succeeded GFS: http://www.pdsw.org/pdsw-discs17/slides/PDSW-DISCS-Google-Keynote.pdf.
