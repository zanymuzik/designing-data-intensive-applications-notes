# Chapter 2 - Data Models 
* A **data model** defines how data is organized and persisted in a system.
* They are fundamentally a layer of abstraction. The choice of data model provides a unified API, one that is good for certain things and bad for others and which hides the implementation and performance details lower down.


* The **relational model** is the most standard data model. Classical relational databases follow this model. In the relational model data is organized in terms of relations (tables) and tuples (rows).
* A **key-value store** is an alternative construction, where each element has a unique key corresponding with some chunk of data as the store.
* Technically speaking the "value" in key-value can be any abstract type, from a string all the way to binary blob. The **document model** is a subtype of key-value stores in which each of the data chunks is forced to be a document.
* An example of a key-value store is Redis. An example of a document-oriented database (a database using the document model) is MongoDB.
* The **graph model** is a data model that treats data as blobs of information connected by edges representing relationships.


* Some relational databases may implement document model support as well, on the side.
* Graph databases can be implemented using relational databases, or key-value stores, or some other underlying system.
* Remember: onion-like API layers!
* The biggest issue with the relational model that so-called "NoSQL" models try to address is **object-impedence mismatch**. The structure of the data in the database doesn't look that much like the structure of the data as it is applied in the program, forcing the programmer to have to deal with two different memory structures.


* For a broad range of applications, document databases are good at locality. Data that is consumed together is located close together within the document schema. In a relational model by contrast useful data can often be several joins away, unless you start denormalizing.
* The strength of relational databases is in joins. If your application needs to perform joins often, then relational models are still the gold standard. Document stores do not intrinsically support these joins so you will have to reimplement them at the application level. If you don't need to perform joins very often, non-relational models are more viable.
* Document databases are more flexible than relational databases because they are **schema-on-read**, whereas relational databases are **schema-on-write**. This just-in-time schema dependency can be a boon or a burden, depending on your product and development cycle.
* When a modification to the data is made, document databases typically have to reload the entire document. If there are many write operations, document database performance suffers. When using a document store you want to have small documents that are consumed nearly completely and modified in large segments.

* Graph databases are implemented using either a property graph or a triples approach. The former is a direct translation of graph theory, while the latter is extracted from work on the semantic web.
* The property graph is based on nodes (storing data in key-value pairs) with sets of edges leading towards and away from them.
* Edges are similarly key-value pair stores, which additionally have a label property as well as (optionally) information on direction: what node is the head and what node the tail (e.g. directed graph or undirected graph?).
* You can naively implement a graph database yourself by creating two relational tables, one with edges and one with nodes. Connections are just foreign keys!
* Triple stores encode these same relationships in a different manner, as sentences of a subject-predicate-object form.
* The agreed-upon data schema is RDF in semantic web circles is RDF.
* The typical canonical form of RDF is in XML, which, no, so you want to use some other way of expressing it.
* SPARQL and Turtle are two general-purpose ways of writing queries against data using the tripe-store model.
* There is also Cypher, which was created for the Neo4j database and semi-limited to it.
#Chapter 3 --- Storage and Retrieval
## Logs and hash maps

* Strictly speaking, a **log** is any append-only sequence of records. Logs may be human-readable, or they may only be machine-readable.
* This is distinct from application logs, which are a specific type of log. The two concepts are sometimes conflated however.
* Logs are a fundamental structure in a database. Logs are fast because it's hard to beat the performance of simply appending to a file.
* Another fundamental structure is the **index**. An index is a data structure which speeds up read operations. It usually slows down writes however because it requires performing work on every insert or append operation.


* The **hash map** is a simple but effective index structure.
* Both Python and JavaScript objects are ultimately just implementations of hash maps. So are the `dict` and `Map` classes in these languages.
* Databases using logs for primary storage are said to be using **log-structured storage**.
* If you combine a log and a hash map, you have a database! Thus the simplest possible practical (you can technically omit the index, but then a read operation would require a full file scan!) database: an append-only file store with an in-memory hash map index pointing to the byte offsets of records of interest.
* [Bitcask](https://en.wikipedia.org/wiki/Bitcask) is basically this, and it's a backing store for a number of more advanced architectures, as well as a database option in and of itself.
* This architecture has lots of advantages:
  * Low read and write latency, with predictable timing. Reading is just a single disc read, writing is a single disc write.
  * High throughput.
  * Backup is as easy as copying the file.
  * Straightforward error recovery. Only the last element being written can get corrupted.

* Of course, it has disadvantages as well:
  * The index must fit in memory (otherwise performance plummets).
  * Range queries get no speed-up (which e.g. time-series data would benefit from).
 
 
* If log-files are left untouched, they will grow to infinity in size.
* In practice you can split your log into *segments* of a certain size, and then periodically **merge** and **compact** those segments into a merged log.
* The compactor iterates through the log file and, for every key written to, finds the most recent value written. These operations are kept in the log, and as part of the merge all previous writes are discarded.


* There are some additional considerations when implementing log files.
* You will want to **compress** the log using a binary encoding scheme.
* A special value called a **tombstone** needs to be appended to the log in the case of key deletion.
* You can recover from a volatile memory crash in the hash map by reading from the start of the log segments, but this can take a long time if the segments are large. You can speed this up by keeping a snapshot of the hash map backed up to the disc.
* You can recover from a database write crash by calculating a checksum on the data segment being written ahead of time, and making sure, when recovering, that the check of the data included in the last write matches your checksum.
* Log files are sequential append-only, so they support multiple reads, and hence concurrent requests, if you want to service that.


* If we insert data into the log in sorted order we can maintain a much smaller log by overwriting prior value assignments in-place. Why don't we do that?
* Mainly because insertion in the middle of a file is a **random write** operation, while appending to the end of the file is a **sequential write** operation.
* In a random write we seek-write-seek-write.
* In a sequential write we seek once, then write multiple times (append from the same cursor).
* The latter is always faster than the former as disc seeks are extremely expensive. Since writing to the log has to happen every single transaction, this time difference matters a lot, and makes log-writing the more attractive option.
* Additionally, the simplicity of the log file makes concurrency and crash recovery much easier to implement.

## SSTables and LSM-trees

* The **SSTable** (Sorted String Table) is a more complex log-structured storage implementation.
* In SSTables we still have log segments, but their contents are now sorted by key.
* This design employs a balanced in-memory data structure, in this context referred to as the **memtable**. The memtable needs to retain key sort order.
* Every time a write or modify operation occurs, the data is written to the memtable.
* The memtable is periodically flushed in key order into a new log segment. This is still a sequential write!
* Those segments are periodically compacted and merged. The difference is that since the segment keys are in sorted order, this process is greatly expedited: we no longer need to scan through every value in every segment, as we can skip over stale key modifications completely!
* Because we no longer write to the log right away, and instead buffer using an in-memory data structure, we must keep an additional unordered log file in play, one that gets written to whenever the memtable gets written to. This is so that the memtable can be repopulated in the event of a crash.


* One advantage of this design is, as previously mentioned, that compacting segments is now much more efficient.
* Another advantage is that you no longer retain a complete index in memory. So this design works with large databases whose hash table index wouldn't fit in memory.
* If all data values were the same size, we could look up values by performing a binary search. However, in practice they are not.
* A sparse in-memory index is retained. At read time this index is queried for the range of memory values a database element could be located at, by finding the immediately smaller and larger keys in the index and taking the range of values in between their endpoints. That range is then scanned to find the value.
* For maximum read efficiency, keep the ranges down to the hardware page size.
* The process of actually scanning the data, once you have the pointer, is extremely fast.
* One more practical modification comes from the observation that with current hardware, disc I/O is more precious than CPU cycles.
* So most databases compress the index address space reference blocks. This makes the data smaller and leads to less time spent reading from persistent storage to volatile memory, but also some CPU cycles expended on decompression.


* Detour note: the memtable in the SSTables implementation needs to be a data structure that affords random insertion order and sorted read order. There are tree structures that have good performance for these tasks, namely red-black trees and AVL trees. I implement these myself in other notebooks in this repo.


* This arrangement is what LevelDB and RocksDB, among other databases, use.
* Cassandra and HBase use a similar scheme derived from the Google Bigtable paper.
* Lucerne, a full-text search indexing engine used by Elasticsearch and Solr, uses a similar method for storing its (much more complicated) term dictionary.


* For sufficiently small amounts of data, a log-structured hash table database is faster.
* But SSTables has several advantages. The most important ones being much larger practical memory limits, due to the sparse in-memory structure, and intrinsic support for range queries.
* There are many potential performance tuning measures that can be taken.
* For example, the SSTables architecture can be slow when looking up keys that do not exist in the database, or which have not been written to in a long time, because walking back the memtable and the list of SSTables takes time.
* You can use a **Bloom filter** (note: implemented in another notebook) to speed this process up. This algorithm is great for approximating set contents.


## B-tree

* The most common database implementation, and the one used by most of the "classic" SQL engines, isn't a log storage design, it's a **B-tree** design.
* The B-tree is a data structure which provides balanced in-order key access. It's not dissimilar to the red-black trees, actually.
* Note: implemented elsewhere.
* B-trees are organized in terms of **pages**. Each page contains references to pages further down the list, except for the last page (the leaf page), which inlines pointers to the data it references.
* Pages are traditionally 4 KB in size, and are designed to emulate the way that memory pages in the underlying hardware work (keeping B-tree pages smaller than hardware pages speeds up access!)
* To look up a value, you start at the root page, whose keys describe a range of values indexed by sort order. You find the key-value pair associated with the index range your value is in. The value will be a pointer to another page, with another range of valid values. Keep burrowing until you get to a leaf page, which contains an exact index location key and a pointer to your desired value. Tada!
* To insert a value, find the legal space for it amongst the leaf pages. If any of the pages along the way are too full to accept new values, split them into half pages, and refactor references appropriately.
* Deleting a value is a lot more involved, however.
* B-trees are balanced trees: for $n$ keys they have a height of $O(\log{n})$.
* B-trees operations are not intrinsically safe. To make them safe you need to add a **write-ahead log**, and write the B-tree operations that are necessary to that log before performing the actual operations.
* In contrast to the file append-only operation of the log-structured database, B-tree databases need to do seek-writes to specific memory pages, which is slower. On the other hand, they do not need to perform the background write operations necessary in log-structured databases.
* Additionally, if you allow for concurrent access (and concurrent modification), you need to introduce locks (specifically, latches on the level of the tree being modified) in order to prevent ongoing writes from causing ongoing reads to return inconsistent data. SSTable designs are much less complex in this regard.
* Some B-tree optimizations:
    * Instead of overwriting pages and maintaining a write-ahead log you can use a copy-on-write scheme: write a new page with the included modification elsewhere in memory, atomically swap the reference from the old page to the new page, and delete the old page. This improves concurrenct read performance, at the cost of write performance. LMDB is an example of a database that implements this.
    * In higher-level pages, instead of storing entire keys store the leading bytes of the keys. Keys only need to provide enough information to boundary where to go next, and smaller keys allow fewer more packed memory pages, a higher branching factor, and ultimately shallower value access. This is known as the **B+ tree**.
    * B-trees try to store their leaf memory pages in sequential order, obviating the need to seek (for certain access, and only when the hardware complies).
    * Pointers may be added to leaf pages pointing to their immediate left and right companion leaf pages. This increases range query speed as it obviates the need to return to the parent page.
    * Finally there are more complex B-tree variants, like **fractal trees**, which attempt to further reduce seek volume.
    
    
* Comparing B-trees and LSM-trees...
* B-trees involve seeks, so they're slower when read volume is high. This is the primary thing that makes LSM-trees more attractive.
* On the other hand, they do not involve periodic compaction. Compaction uses slow disc I/O resources. It's possible to schedule it when nothing else is hitting the disc, but hard to do so reliably. This means that, particularly at higher percentiles, B-trees have more reliable performance than LSM-trees do.
* In general it's hard to predict which of these two structures will perform better on a particular workload. You have to test emperically.

## More index considerations
* Secondary indices are barely different from primary indices. The only implementation differene is that they allow duplicate keys, so you need to bake a slightly different key. An easy way around this is to append the row number.


* The key in a primary index is always an identifier for the thing you are searching for.
* The value however could be one of two things. Either it's exactly the data in question, or a pointer to that data somewhere else in memory.
* The latter is the **heap file** approach.
* The heap file approach is more common. If multiple indices are defined on a particular chunk of data, it avoids duplicating it.
* An implementation detail is what to do when inserting new data.
* If the new data is smaller than or the same size as the previous data (as it would be when you have a fixed-width type!), you can overwrite in place.
* If the new data is bigger, you need to either write to a new location and move the reference, or write to a new location and populate a pass-through pointer.
* The jump to the heap location is a seek, so it has a time cost. Storing the data directly in the index instead is known as a **clustered index** approach, and it avoids this cost.
* Secondary indices in a table with a clustered index "just" populate references to the primary index.
* However, a clustered index represents duplication of data, since obviously that data has to exist in the log first. Thus you trade worse write performance for better read performance.
* Also, it hits size limits much faster, if your objects are large.
* A compromise between the two is a **covering index**, which stores just *some* of the columns in the index. Write operations on these columns are slower, but read operations are faster. Columns not "covered" are faster.


* Finally, there are multi-column indices. These speed up queries that pivot on several columns at once.
* The most common type of multi-column index is the **concatenated index**, which is keyed using a sequence of values. The semantics are slightly different, but the structure is mostly the same.


* There are more complex index structures, like R-trees and whatever the text search engines use. But they're slightly out of score for now! Something to investigate further down the line.

## OLAP systems
* Databases in production can generally be split into two access patterns. **OLTP** systems (short for online transaction processing) is the original use-case. **OLAP**, short for online analytics processing, is the new use case.
* The operational reasons for splitting OLTP from OLAP I know too well!
* OLAP systems are less creative than OLTP systems in terms of overall design. Almost all applications use some variant of the **star schema**.
* In the star schema there is a central table that acts as an access point, which provides foreign keys to a bunch of other tables.
* Each of the other tables is known as a **fact table**, and it provides details about one specific aspect of the elements in the central table: who, what, where, when, how, why?
* A more complex and normalized variant of this schema is the **snowflake schema**, which has additional branching factors on the fact table.
* These arrangements are so named because they look a star or a snowflake surrounding a central fact table in the middle.


* OLAP systems injest data every once in a while. Simultaneously, the queries that are run against them are often very heavy, touching hundreds of thousands and potentially billions of rows.
* One way to increase query efficiency is to maintain a **materialized view** of oft-used summary statistics. This is known as a **data cube**.
* In a data cube you choose a certain number of dimensions $n$ (an $n$-fold data cube), and then compute a summary statistic against the resultant aggregate.
* If those statistics are used often, it may be worth paying that cost up-front once at injestion time, instead of having to rebuild every time a query using that summary comes in.


## Column-oriented storage

* While the OLTP access pattern is row oriented, the OLAP access pattern is column-oriented. For efficient OLAP architecture, you want to provide locality on the columns, *not* the rows.
* Thus the natural OLAP adaptation: **column-oriented stores**.
* Column-oriented databases fundamentally lay colums contiguously in memory, instead of rows.


* Additionally, ordering data in terms of columns unlocks columnar compression.
* There are various schems that you can use. One of the most common is the **bitmap encoding**, which calculates a dummy encoding for each product and stores it.
* When there is low columnar cardinality, bitmaps are very efficient memory-wise. They are also essentially **vectorized** already, so math operations (like selection with `OR` clauses) are extremely fast, making common queries much faster
* You can compress further using something along the lines of **run-length encoding**.
* Compression is particularly important in OLAP systems because these may need to scan millions of entries at a time. Sequential disk I/O is the bottleneck, not entry look-ups, so reducing the memory footprint is crucial.
* Compression is a whole other topic worth exploring.


* You still insert data into column stores in a row-wise manner.
* It probably doesn't make sense to use B-trees to store the data, as overwriting data with new values that are too large to fit in the allotted memory space will break column contiguity.
* Log structured storage still works well, however. (need to think some more about why this is the case)


* When data is organized in column order, there is no intrinsic row sort order.
* Leaving the rows unsorted improves write performance, as it results in a simple file append.
* Sorting the rows improves read performance when you *do* need to query specific rows in a column-oriented database. You can multi-sort by as many rows as desired, obviously, but rows beyond the first will only help when performing grouped queries.
* Additionally, sorting keys can help greatly with compression. Especially on the first sort-order key, something like run-length encoding can result in incredible read performance. With low enough cardinality multiple gigabytes of data can get pushed down to mere kilobytes in size!
* A clever idea is to actually maintain multiple sort orders on disc, by replicating the data with several different sorts. Obviously this is taking read performance at the cost of write performance as far is will go. Vertica is one example of a database that offers this feature.
#Chapter 4 --- Encoding and Evolution

## Introduction to data serialization
* Applications evolve over time.
* When the application is server-side, you perform a **rolling upgrade**, taking down a few nodes in your deployment at a time for upgrades.
* When the application is client-side, updating is at the mercy of the user.
* To make this easier, on the data layer, you may have either or both of **backwards compatibility** and **forwards compatibility**. The latter is far trickier.


* In memory we consider data in terms of the data structures they live in.
* On disc we work with memory which has been **encoded** into a sequence of bytes somehow.
* With a handful of exceptions for memory maps or memory-mapped files, which are direct representations of on-disc packing in in-memory data structure terms.


* The process of transliteration between these two formats is known as serialization, encoding, or marshalling. A specific format is a **data serialization format**.
* Most languages include language-specific formats, like `pickle`, which can be used to store language object.
* These formats are easy to use when working with a specific language, but how serializability boundaries and are not easily compatible with other languages.
* Still, if your data always stays inside of your application boundary, these formats are fine.


## Human-readable data interchange formats
* JSON and XML (and CSV, and the other usual suspects) are common **data interchange formats**, meant to be moved between application boundaries.
* These formats are considered to be lowest common denominators, however.
* They have parsing problems. For example, it's often impossible to difficult to determine the type of an object.
* Being human-readable, they are also inefficient in resource terms when performing network transfers.
* Still, for simple use cases these formats are usually sufficient.


## Binary data interchange formats
* An improvement on the human-readable data interchange formats are encoded binary formats.
* Several competing encodings of the human-readable data interchange formats above exist. For example, MongoDB stores JSON data encoded in the BJSON format.
* Your application can also invent its own binary file format, if it so desires.
* For general-purpose use and interchangabilty, however, several binary data serialization formats exist.
* The two examples the book uses are Google Protobufs and Apache Thrift (both of which are still in good use in the ecosystem today).
* Binary data interchange formats provide a wealth of tooling, including programs that can be run to automatically machine-generate APIs for working with the data in your language of choice.
* These APIs are naturally verbose and do not match very well against common language patterns, becuase they are machine-written and not human-written, but they work well enough.
* These advanced binary data interchange formats are especially neat in that they provide forward and backwards compatibility built-in.
* In the context of data interchange formats this is known as **schema evolution**.


* The book talks about two similar but differently designed binary interchange formats. Apache Thrift and Google Protocol Buffers are in one camp, and Apache Avro is in the other.


* Fields in Protobuf (and in Thrift) are identified by field IDs.
* These field IDs can be omitted if the field is not required, in which case they will simply be removed from the encoded byte sequence.
* But once assigned, they cannot be changed, as doing so would change the meaning of past data encodings.
* This provides backwards compatibility. There is one catch however. You cannot mark new fields required, for the obvious reason that doing so would cause all old encodings to fail.
* How about forward compatibility? Every tag is provided a type annotation. When an old reader reads new data, and finds a type it doesn't know about, it skips it, using the type annotation to determine how many bytes to skip. Easy!
* I have some experience with Google Protobufs, so I know how this whole thing works reasonably well.


* Aother perspective, used by Apache Avro, is that these formats must be resiliant to differences between the **reader  schema** and the **writer schema**. The challenge is to have a reader that understands every possible version of the writer.
* Avro requires you provide version information on read, which the other two formats do not require. This is additional overhead, as you must either encode that information in the file or provide it through some other means (the former is better if the file is big, and the latter if the file(s) are small).
* This allows Avro to omit data tags. This in turn makes Avro much easier to use with a dynamic schema, e.g. one that is changing all the time.
* This use case is what motivated Avro in the first case. And this is a tradeoff! Avro is more dynamic, Buffers and Thrift are more static but less work.


* All three are equipped with **interface description languages**. These allow you to perform **code generation** and get a machine-written API for your data
* However, code generation is mainly useful for statically typed languages, which benefit from explicit type checking. Dynamically typed languages, like Python, do not get much benefit.


## Data flow
* The rest of the chapter discusses the general idea of **data flow**.
* Data flow is how data flows through your system. It involves thinking about data usage patterns, application boundaries, and similar such things.
* The book points to three types of data flows in particular.

* Databases are the first data flow concept.
* Both backwards and forward compatibility matters in a database. Backwards compatibility is important because a database is fundamentally messages to your future self. Forward compatibility matters because when you perform rolling updates, your database will be accessed concurrently by both newer and older versions of software.
* When you perform a database migration, the format of the underlying data is actually left unchanged, in those cases where no new information is stored (e.g. adding a new column full of `null` values). Only when you touch the newly created columns will the database figure out where to store the data so that it has space to include the additional information.
* This is in recognition that the data that gets stored typically outlives the code that stores it, and your database may have values in it that have not been touched in five years or more! Moving all of that at once is expensive, so databases migrate lazily when they can.
* An approach that deals data principally through databases is using what is known as an **integration database**. Heavy-on-database data flow is an architectural pattern commonly associated with **monolithic architecture**.


* The second generalized type of data flow is service communication.
* In service communication you have services that talk to one another using some kind of agreed-upon interchange format. The web is a great example.
* Often these services are organized in terms of clients and servers, with clients talking to the servers on behalf of end users.
* API calls over a network of some kind have a long lineage.
* On the web, this is where REST and SOAP live.
* This is where **service-oriented architecture** and **microservices** matter (the latter is a more recently coined and more specific subset of the former).
* REST is a design philosophy that opines on how well-designed services built over `HTTP` and `HTTPS` should look like. SOAP, by contrast, is an XML-based and technically HTTP-independent (but usually HTTP-using) design philosophy. The two compete for mindshare.
* REST is winning over SOAP, at least in part due to the decline of XML.
* The other philosophy is **remote procedure calls**, or RPC. RPC wants you to treat your inter-service calls the same way you would treat your intra-service calls (e.g. data flow *within* a program).
* RPC is a nice idea but it has problems. From a design perspective it's important to understand what they are:
  * A local function call is predictable, in the sense that it succeeds or fails or causes the program to crash. A remote call may simply never result in a response (until you time it out yourself).
  * Moreover, remote calls are basically always way less durable and have more variable latency, because they rely on potentially flakey, bandwidth-limited network.
  * If you retry an operation you may duplicate an action on the endpoint. This can be worked around but requires additional thought and design.
  * Networks have a much higher fixed cost than local functions. Youy need to encode all of the data necessary and send it out. Additional serialization is necessary, and large objects are a problem.
* At the end of the day, costs of space in memory is much lower than it is in-network. A good term: good local calls are fine, network calls are coarse. This is a big reason why monoliths are hard to refactor into microservice things!


* The third type is the message bus pattern.
* Your applications can communicate with one another using streams of messages that get passed through a **message broker**.
* Example message brokers are ZeroMQ, Apache Kafka, and Google PubSub.
* When using this pattern, your application parts no longer need to be aware of one another. Message emitters can emit messages without caring about who's listening, and message listeners can consume messages without being aware of who's emitting them.
* This pattern is good for separation of concerns. The trade-off is that there is one more service you have to maintain. Message brokers are not a lightweight solution overhead-wise.
# Chapter 5 --- Replication
* **Replication** is one of the strategies for distributing data processes across multiple nodes (the other is **partitioning**, the subject of a later chapter).
* The difference between replication and partitioning is that whilst replication has you keep multiple copies of the data in several different places, partioning has you split that one dataset amongst multiple partitions.
* Both strategies are useful, but obviously there are tradeoffs either way.


* Replication strategies fall into three categories:
  * **Single-leader** &mdash; one service is deemed the leader. All other replicas follow.
  * **Multi-leader** &mdash; multiple services are deemed leaders. All other replicas follow.
  * **Client-driver** &mdash; not sure.
  
## Single-leader replication

* The first important choice that needs to be made is whether replication is synchronous or asynchronous.
* Almost all replication is done asynchronously, because synchronous replication introduces unbounded latency to the application.
* User-facing applications generally want to maintain the illusion of being synchronous, even when the underlying infrastructure is not.
* Asynchronous replication introduces a huge range of problems that you have to contend with if you try to maintain this illusion.
* (personal note) GFS is an interesting example. GFS was explicitly designed to be weakly consistent. That unlocked huge system power, and the additional application layer work that was required to deal with the architecture was deemed "worth it" because it was on the engineer's hands.


* The precise configuration of concerns with a single-leader replication strategy differs. At a minimum the leader handles writes, and communicates those writes to the follower, and then the followers provide reads.


* If a follower fails, you perform catch-up recovery. This is relatively easy (example, Redis).
* If a leader fails you have to perform a **failover**. This is very hard:
  * If asynchronous replication is used the replica that is elected the leader may be "missing" some transaction history which occurred in the leader. Totally ordered consistency goes wrong.
  * You can discard writes in this case, but this introduces additional complexity onto any coordinating services that are also aware of those writes (such as cache invalidation).
  * It is possible for multiple nodes to believe they are the leader (Byzantine generals). This is dangerous as it allows concurrent writes to different parts of the system. E.g. **split brain**.
  * Care must be taken in setting the timeout. Timeouts that are too long will result in application delay for users. Timeouts that are too short will cause unnecessary failovers during high load periods.
* Every solution to these problems requires making trade-offs!


## Replication streams

* How do you implement the replication streams?


* The immediately obvious solution is **statement-based replication**. Every write request results in a follower update.
* This is a conceptually simple architecture and likely the most compact and human-readable one. But it's one that's fallen out of favor due to problems with edge cases.
* Statement-based replication requires pure functions. Non-deterministic functions (`RAND`, `NOW`), auto-incrementation (`UPDATE WHERE`), and side effects (triggers and so on) create difficulties.


* **Write-ahead log shipping** is an alternative where you ship your write-ahead log (if you're a database or something else with a WAL).
* This is nice because it doesn't require any additional work by the service. A WAL already exists.
* This deals with the statement-based replication problems because WALs contain record dumps by design. To update a follower, push that record dump to it.
* The tradeoff is that record dumps are a data storage implementation detail. Data storage may change at any times, and so the logical contents of the WAL may differ between service versions.
* The write-ahead logs of different versions of a service are generally incompatible! But your system will contain many different versions.
* Generally WALs are not designed to be version-portable, so upgrading distributed services using WAL shipping for replication requires application downtime.


* **Logical log replication** is the use of an alternative log format for replication. E.g. replication is handled by its own distinct logs that are only used for that one specific purpose.
* This is more system that you have to maintain, but it's naturally backwards and forward compatible if you design it right (using e.g. a data interchange format like Protobufs) and works well in practice.


* Post-script: you may want partial replication. In that case you generally need to have application-level replication. You can do this in databases, for example, by using triggers and stored procedures to move data that fits your criteria in specific ways.


* Asynchronously replicated systems are naturally **eventually consistent**. No guarantees are made for when "eventually" will come to pass.
* **Replication lag** occurs in many different forms. Some specific types of lag can be mitigated (at the cost of performance and complexity), if doing so is desirable for the application (but you can only recover a fully synchronous application by being fully synchronous).
* The book cites three examples of replication lag that are the three biggest concerns for systems.


* The first one: your user expects that data they write they can immediately read. But if they write to a leader and read from a replica, and the replica falls behind the leader, they may not see that data.
* A system that guarantees users can see their own modified data on refresh is one which provides **read-your-write consistency**. In practice, most applications want to provide this!
* You can implement read-your-writes in a bunch of different ways, but the easiest way is to simply have the leader handle read requests for user-owned data.


* Another issue occurs if a user is reading from replica, and then switches to another replica that is further out of sync than the previous one. This creates the phenomenon of going backwards in time, which is bad.
* Applications which guarantee this doesn't happen provide **monotonic reads**.
* An easy way to get monotonic reads is to have the user always read from the same replica (e.g. instead of giving them a random one).


* The third kind of inconsistency is **causal inconsistency**. Data may get written to one replica in one order, and to another in a different order. Users that move replicas will see their data rearranged.
* Globally ordered reads require that the system be synchronous, so we can't recover all of the order. However, what we can do is gaurantee that events that have a causal relationship (e.g. this event happened, which caused this event to happen) *are* mirrored correctly across all nodes.
* This is a weaker gaurantee than fully synchronous behavior because it only covers specific subsequences (user stories) from the data, not the entire sequence as a whole.
* A system that handles this problem is said to provide **consistent prefix reads**.


* It's notable that you can avoid all of these problems if you implement **transactions**. Transactions are a classical database feature that was oft-dropped in the new world because it is too slow and non-scalable for a distributed system to implement. They seem to be making a comeback now, but it's still a wait-and-see.


## Multi-leader replication
* Multi-leader replication is a more complex but theoretically more highly available architecture.
* If there are multiple leaders then if one leader goes down a failover is not necessary (as long as *all* of the leaders don't go down, and the remaining leaders can handle the traffic).
* In addition to the additional durability, there are also various advantages in terms of latency and robustness in the face of network partitions.
* However due to implementation complexity they tend to only be implemented for multi-datacenter operations.
* The fundamental new problem that multi-leader architectures introduce is the fact that concurrent authoritative writes may occur on multiple leaders.
* Thus your application has to implement some kind of **conflict resolution**. This can occur on the backend side, or it may be implemented on the application side. It may even require user intervention or prompting.
* Interestingly, applications which sync data between clients but whose clients are intended to work offline are technically multi-leader, where each leader is the individual application. E.g. calendar applications.

## Client-driven replication
* This is the last of the three main architectures.
* In this case you have "smart clients", e.g. clients that are responsible for copying data across replicas (or alternatively, designated nodes that do this on behalf of the clients).
* Client-driven replication were for a long time not a particularly popular strategy because it requires establishing lots of network connections between the client and the replicas. It's what Dynamo used (though it's not what DynamoDB, the hosted product, uses), and that's made it popular. Cassandra does this form example.


* How do you handle data replication without designated leaders?
* Services using client-driven replication do so using two processes.
* A **read-repair** occurs when the client asks for data, gets it, finds that some of the nodes are responding with old data, and forwards a "hey, update" to those nodes.
* An **anti-entropy process** constantly or ocassionally scans for inconsistencies in the data nodes, and repair them.
* Clients read using **quorums** of certain sizes, taking the most recently updated data point from amongst the nodes that respond to a request. Quorums offer a tunable consistency-speed tradeoff. Larger quorums are slower, as they are tied down by the speed of the slowest respondent, but more consistent. Smaller quorums are faster, requiring fewer node responses, but less consistent.


* A difficulty that arises with this architecture is what to do is the nodes that are normally responsible for a quorum write are unavailable.
* The strategy is to perform a **sloppy quorum**: write that data to the right amount of still-available nodes.
* Once the nodes that are the true home for that data become available again, perform a **hinted handoff** to those nodes.
* Reader note: the great AWS EC2 node failure outage of 2013 or thereabout occurred because this mechanism floored the network.
# Chapter 6 --- Partitioning

* **Partitioning** is the splitting of data across multiple nodes.
* It is a necessity for scaling to particularly large datasets.
* It is usually combined with replication.


* The main problem that needs to be addressed by your partitioning scheme is the avoidance of **hot spots**.
* Data is not accessed or written to equally. If you partition too many commonly worked-with pieces of data to the same node, your performance will degrade.


## Partition types

* Depending on your goal, there are a couple of ways to partition data that are currently in use.
* The first is **partitioning by key range**. Partitioning by key range is simple: you just push contiguous strips of data to the same partitions.
* If there are hot spots in a particular key range, your performance will suffer.
* On the other hand, key range partitions support efficient range queries, as all of the data in a range will land on the same node. The other scheme does not provide this property.
* The alternative is **partitioning by key hash**.
* A good hashing function will uniformally distribute skewed data, "smudging" your data evenly across your partitions.
* This approach means no hot spots. However, you lose range query performace, as these must now hit all of the nodes.
* Cassandra has an interesting compromise position. It allows compound primary keys, and performs partitions by key hash based on the first key value only. All of the values from the second key element on will land on the same partition, so these keys can still be ranged over effectively!
* Note that even with the key hash technique you can still have hot spots, if your application has Zipf-like elements in it. For example, Twitter uses different serving techniques for celebrities then it does for "the rest of us".


## Partitioning secondary indices

* Partitioning support with secondary indices is another subject matter.
* Because of how difficult this is to implement, some data stores that support partitioning omit secondary indices entirely!
* Generally there are again two approaches.
* One is **scatter-gather**. The secondary indices are stored locally alongside the partitions. Requests that mean to use the secondary index must hit *all* of the nodes in order to stitch together the full index.
* This is also known as the **local index** approach.


* The alternative is a **global index**. The global index stores all of the information on all of the data.
* This global index isn't kept on one node, as doing so would create a hot spot and defeat the entire purpose of the thing. Strategies for storing it differ, but it's generally dealt with independently of data duplication.
* A local index requires hitting all or many of the partitions when performing an index-assisted read, but doesn't affect write performance. A global index requires hitting only a node containing the global index, but slow down writes, as these must now visit that node to notarize inserts.
* In practice global indices are updated asynchronously, which avoids this slowdown. But this introduces...all of those problems into the equation. Exciting.


## Rebalancing

* If you partition, you will eventually need to **rebalance**.
* Rebalancing schemes are all about accounting for the introduction of additional partitions into the store, whilst limiting the amount of actual data that needs to get moved around in response.
* E.g. a `hash mod N` would work poorly because it would result in a lot of data having to be moved.
* There are two popular rebalancing schemes.


* One is to have a **fixed partitions strategy**, where that number greatly exceeds the number of actual nodes. Cassandra calls these "virtual nodes" for example.
* When a new partition comes up, the partition grabs responsibility from some still-mostly-empty partitions from the existing partitions. Little data moves!
* This is an additional abstraction layer, however, and it requires operational overhead. The point is lost if you use too extreme a number of partitions.


* The laternative is **dynamic partitioning**. In this case you specify a maximum partition size, and split the partition when the data approaches that size (like growing a `dict` in Python).
* Partitions can stay on the same node after a split for a while, but will transfer over to a different node (snapshot operation). E.g. HBase implements this strategy using HDFS snapshotting.
* The advantage of this scheme is that it requires. The disadvantage is that it is ungainly when the datasets are small.


## Service discovery

* This chapter briefly mentions the service discovery problem, posing three possible solutions:
  * The client can connect to any node, which can forward the request through the network to the node that has the requested data.
  * A routing tier can connect to the necessary node.
  * The client can maintain information on the right node, and connect to it directly.
* Zookeeper is a hierarchical key-value store designed to store config metadata that is used by many services to address this exact problem.
# Chapter 7 --- Transactions

* Transactions provide guarantees about the behavior of data that are fundamental to the old SQL style of operation.
* Transactions were the initial casualty of the NoSQL movement, though they are starting to make a bit of a comeback.
* Not all applications need transactions. Not all applications want transactions. And not all transactions are the same.

##  ACID

* **ACID**, short for atomic-consistent-isolated-durable, is the oft-cited headliner property of a SQL data store.
* Operations are **atomic** if they either succeed completely or fail completely.
* Consistency is meaningless, and was added to pad out the term and make the acronym work.
* Operations are **isolated** if they cannot see the results of partially applied work by other operations. In other words, if two operations happen concurrently, any one will see the state either before or after the changes precipitated by its peer, but never in the middle.
* A database is **durable** if it doesn't lose data. In practice nothing is really durable, but there are different levels of guarantees.

## Single versus multi object operations
* An operation is **single-object** if it only touches one record in the database (one row, one document, one value; whatever the smallest unit of thing is). It is **multi-object** if it touches multiple records.
* Basically all stores implement single-object operation isolation and atomicity. Not doing so would allow for partial updates to records in cases of failures (e.g. half-inserting an updated JSON blob right before a crash), which is Very Bad.
* Isolated and atomic multi-object operations ("strong transactions") are more difficult to implement because they require maintaining groups of operations, they get in the way of high availability, and they are hard to make work across partitions.
* So even though distributed data stores that abandon transactions may offer operations like e.g. multi-put, they allow these operations to result in partial application in the face of failures.
* These services work on a "best effort" basis. If an error occurs it is bubbled up to the application, and the application layer must decide how to proceed.
* Can you get away without multi-object operations? Maybe, but only if your service is a very simple one.
* All of the things that follow are **race conditions**.


## Weak isolation levels
* The strongest possible isolation guarantee is **serializable isolation**: transactions that run concurrently on the same data are guaranteed to perform the same as they would were they to run serially.
* However serializable isolation is costly. Systems skimp on it by offering weaker forms of isolation.
* As a result, race conditions and failure modes abound. Concurrent failures are really, really hard to debug, because they require lucky timings in your system to occur.
* Let's treat some isolation levels and some concurrency bugs and see what we get.


* The weakest isolation guarantee is **read committed**. This isolation level prevents dirty reads (reads of data that is in-flight) and dirty writes (writes over data that is in-flight).
* Lack of dirty read safety would allow you to read values that then get rolled back. Lack of dirty write safety would allow you to write values that read to something else in-flight (so e.g. the wrong person could get an invoice for a product that they didn't actually get to buy).
* How to implement read committed? 
* Hold a row-level lock on the record you are writing to.
* You could do the same with a read lock. However, there is a lower-impact way. Hold the old value in memory, and issue that value in response to reads, until the transaction is finalized.

* If a user performs a multi-object write transaction that *they* believe to be atomic (say, transfering money between two accounts), then performs a read in between the transaction, what they see may seem anomolous (say, one account was deducted but the other wasn't credited).
* **Snapshot isolation** addresses this. In this scheme, reads that occur in the middle of a transaction read their data from the version of the data (the *snapshot*) that preceded the start of the transaction.
* This makes it so that multi-object operations *look* atomic to the end user (assuming they succeed).
* Snapshot isolation is implemented using write locks and extended read value-holding (sometimes called "multiversion").


* Note: repeatable read is used in the SQL standard, but is vague. Implementations like to have a reapatable read level, but it's meaningless and implementation-specific.


* Next possible issue: **lost updates**. Concurrent transactions that encapsulate read-modify-write operations will behave poorly on collision. A simple example is a counter that gets updated twice, but only goes up by one. The earlier write operation is said to be lost.
* There are a wealth of ways to address this problem that live in the wild:
  * Atomic update operation (e.g. `UPDATE` keyword).
  * Transaction-wide write locking. Expensive!
  * Automatically detecting lost updates at the database level, and bubbling this back up to the application.
  * Atomic compare-and-set (e.g. `UPDATE ... SET ... WHERE foo = 'expected_old_value'`).
  * Delayed application-based conflict resolution. Last resort, and only truly necessary for multi-master architectures.


* Next possible issue: **write skew**.
* As with lost updates, two transactions perform a read-modify-write, but now they modify two *different* objects based on the value they read.
* Example in the book: two doctors concurrently withdraw from being on-call, when business logic dictates that at least one must always be on call.
* How do you mitigate?
* This occurs across multiple objects, so atomic operations do not help.
* Automatic detection at the snapshot isolation level and without serializability would require making $n \times (n - 1)$ consistency checks on every write, where $n$ is the number of concurrent write-carrying transactions in flight. This is way too high a performance penalty.
* Only transaction-wide record locking works. So you have to make this transaction explicitly serialized, using e.g. a `FOR UPDATE` keyword.


* Next possible grade of issue: **phantom write skew**.
* A `FOR UPDATE` will only stop a write skew if the constraint exists in the database. If the constaint is *the lack of* something in the database, we are stuck, because there is nothing to lock. Example of where this might come up: two users registering the same username.
* You can theoretically insert a lock on a phantom record, and then stop the second transaction by noting the presence of the lock. This is known as **materializing conflicts**. 
* This is ugly because it messes with the application data model, however. Has limited support.
* If this issue cannot be mitigated some other way, just give up and go serialized.


## Serialization implementation strategies
* Suppose you decide on full serializability. How do you implement it? What is so onorous that so many data stores choose to drop it?
* There are three ways to implement serializability currently in the market.


* The first and most literal way is to run transactions on a single CPU in a single thread. This is **actual serialization**.
* This only became a real possible recently, with the speed-ups of CPUs and the increasing size of RAM. The bottleneck is obviously really low here. But it's looking tenable. Systems that use literal serialization are of a post-2007 vintage.
* However, this requires writing application code in a very different way.
* The usual way of writing queries is to incrementally update values as the user performs actions. The actions are performed using application code, so the requisite read-update-write loops require constant back-and-forth with the database. This is known as "interactive querying".
* Truly serial databases only work if you use **stored procedures** instead.
* Stored procedures are database-layer operations that implement arbitrary code on incoming data.
* Databases classically offered their own stored procedure languages, which were trash. Recent entries in the field embed scripting languages like Clojure or Lua (the latter being what Redis uses) instead, making this much more tenable.
* When you lean on stored procedures over application code, you need to design your applications to minimize reads and writes, and to let the logic be performed on the database level (via the stored procedure), not in the application layer.
* This is very possible, but it is very different...
* Partitioning the dataset whilst using the architecture allows you to scale the number of transactions you can perform with the number of partitions you have.
* However, this is only true if your code only hits one partition. Multi-partition code is *order of magnitudes slower*, as it requires network moves!


* **Two-phase locking** is the oldest technique, and for a long time the only one.
* Two-phase locks are very strong locks, strong enough to provide true serialization. A basic two phase lock performs as follows:
  * Transactions that read acquire shared-mode locks on touched records.
  * Transactions that write acquire, and transactions that want to write after reading update to, exclusive locks on touched records.
  * Transactions hold the highest grade locks they have for as long as the transaction is in play.
* In snapshot isolation, reads do not block reads or writes and writes do not block reads. In two-phase locking, reads do not block reads or writes, but writes block everything.
* Because so many locks occur, it's much easier than in snapshot isolation to arrive at a **deadlock**. Deadlocks occur when a transaction holds a lock on a record that another transaction needs, and *that* record holds a lock that the other transaction needs. The database has to automatically fail one of the transactions, roll it back, and prompt the application for a retry.
* Why is 2PL bad? Because it has very bad performance implications. Long-blocking writes are a problem. Deadlocks are a disaster. Generally 2PL architectures have very bad performance at high percentiles; this is a main reason why "want-to-be sleek" systems like Amazon have moved away from them.
* There's one more implementation detail to consider: how 2PL addresses phantom skew.
* You can evade phantom skew by using **predicate locks**. These lock on all data in the database that matches a certain condition, even data that doesn't exist yet.
* Predicate locks are expensive because they involve a lot of compare operations, one for each concurrent write. In practice most databases use **index-range locks** instead, which simplify the predicate to an index of values of some kind instead of a complex condition.
* This lock covers a strict superset of the records covered by the predicate lock, so it causes more queries to have to wait for lock releases. But, it spares CPU cycles, which is currently worth it.


* The third technique is the newest. It is **serializable snapshot isolation**.
* True to its name it is based on snapshot isolation, but adds an additional algorithmic layer that makes it serialized and isolated.
* SSI is an **optimistic concurrency control** technique. **Two-phase locking** is a **pessimistic concurrency control** technique. SSI works by allowing potentially conflicting operations to go through, then making sure that nothing bad happens; 2PL works by preventing potentially conflicting operations from occurring at all. 
* An optimistic algorithm has better performance when there is low record competition and when overall load is low. It has poorer performance when lock competition is high and overall load is high.
* It beats 2PL for certain workloads!
* SSI detects, at commit time (e.g. at the end of the transaction), whether or not any of the operations (reads or writes) that the transaction is performing are based on outdated premises (values that have changed since the beginning of the transaction). If not, the transaction goes through. If yes, then the transaction fails and a retry is prompted.
* For read-heavy workloads, SSI is a real winner!
# Chapter 8 --- Distributed Failures

* Computers are designed to fail all at once and catastrophically.
* Distributed services are fundamentally different. They need to fail only partially, if possible, in order to allow as many services as possible to continue to operate.
* This unlocks a lot of additional failure modes, **partial failures**, which are both hard to understand and hard to reason about.


* Long discussion of network flakiness...


* Modern computers have both a time-of-day clock and a monotonic clock. The former is a user-facing clock that can move backwards or stop in time. The latter is a constantly forward-propagating clock, but only the difference between values is meaningful; its absolute magnitude is meaningless.
* Both clocks are ruled over by NTP synchronization, when the computer is online.
* These clocks are quartz-based, and subject to approximately 35 ms average errors.
* Furthermore, processes may pause for arbitrarily long periods of time due to things like garbage collection and context switches.
* Things that depend on timeouts, like creating a global log in a distriobuted system (databases) or expiring leader-given leases (GFS), must account for this possibility.
* You can omit these delays if you try hard enough. Systems like flight controls and airbags operate are **hard real-time systems** that do so. But these are really, really hard to build, and highly limiting.


* Algorithms can be proven to work correctly in a given an unreliable **system model**. But you have to choose your model carefully.
* A point that Jespen makes: these algorithms are hard to implement in a durable way. The theory is far ahead of the practice, but a lot of architects try to homegrow their conflict resolution, which is problematic.


* One simplifying assumption we make when designing fault tolerant systems is that nodes and services never purposefully lie.
* If a node sends an knowingly untrue message to another node, this is known as a **Byzantine fault**. 
* Byzantine faults must be addressed in the design of public distributed systems, like the Internet or blockchain. But they can be ignored in the design of a service system, as obviously you will not purposefully lie to yourself.


* In designing a distributed system, one that is protected from Byzantine faults, you often need to design around causal dependency. You need to understand what a node knew when it made a decision, as that knowledge informs how you will heal any resulting divergence.
* **Vector clocks** are an algorithm for this (e.g. Dynamo).
* An alterantive is preserving a **total ordering** of operations (e.g. MongoDB).
# Chapter 9 --- Consistency and Consensus

* All durable data systems provide **eventual consistency**, but their implementations of the "eventually" part differ.
* In the pre-previous chapter we examined isolation, which is consistency within (and between) concurrent transactions.
* Two chapters before that we examined broad consistency gaurantees in the context of partitioned data.
* Consistency is a difficult topic, and it's one wrapped up in all of the other concerns of distributed systems.


## Linearizability
* A distributed system is **linearizable** if:
  * All operations are atomic, acting as though they happened at instantaneous moments in time, resulting in an implicit order of operations within the system.
    * Concurrency or an illusion of concurrency may happen, but operations are sequential within the system.
  * All reads and compare-and-sets are up-to-date at return time with respect to that order (no stale operations).
* Not part of this definition: op start times (e.g. when the request is made) do not determine operation order. The atomic operation may occur at any time before the ack. With concurrent ops, the service may determine the landing sequence.


* Seriazation, the strongest isolation gaurantee, and linerazation, here, are two different concepts of service behavior.
* The strongest possible gaurantees a system can provide overall is "strict serializability", which is both of these properties combined.


* Even if your system does not provide linearizability, some system components may still require linearizability to function properly.
* Lock services, leader elections, data uniqueness gaurantees, and two-channel or cross-channel operations (e.g. concurrently asynchronously write a file to a file system and a message to a message broker telling people to do something with that file) all require linearizability.


* Asynchronous multi-leader and leaderless replication are not linearizable due to potential data loss in a failover. Synchronous multi-leader and leaderless replication are linearizable, but you rarely want to pay this performance cost. Single-leader replication can be made synchronous, but again, meh.
* Sloppy quoroms (used in leaderless replication) are obviously not linearizable due to data divergence.
* Strict quoroms (same) are not linearizable if the op is applied immediately on the replica. They are linearizable if the write is delayed until all replicas ack, but this is usually not the default behavior.


* Brief tangent on the CAP theorem, how it's mainly historical nowadays (I still find it to be a useful heuristic but OK).


## Ordering
* Linearizability implies **causal consistency** (discussed earlier).
* That is because linearizability affords a **total ordering**.
* However, you can have causal consistency without linearizability (a **partial ordering**).
* A causally consistent system preserves causally linked orders of operations.
* In practice, however, determining which operations are causally linked and which ones are not is tedious.
* Thus in practice in a causally consistent system we weaken the second criterion of linearizability:
  * All operations are atomic, acting as though they happened at instantaneous moments in time, resulting in an implicit order of operations within the system.
  * If two ops are in flight at the same time, the ops may be applied in any order.
* The difference between causal consistency and linearizability is that in the latter case the timestamps of concurrent requests cannot be compared, because there *is* no globally consistent timestamp.


* Causal consistency can be achived without linearization.
* Causal consistency is the strongest consistency model that doesn't incur a heavy network cost.
* Providing it requires defining a sequence of operations across nodes that the nodes agree upon.
* First problem: how do you get a sequence of numbers that matches the operational sequence?
* Using a naive solution, like clocks, won't work, because see chapter 8.


* **Lamport timestamps** are a lightweight way of achieving this agreed-upon sequential order. In this algorithm nodes increment a `(node_unique_id, node_sequence_number, largest_sequence_number_seen)` counter.
* If a node wants to add an op to the log, it increments its sequence number.
* If a node sees a message with a larger sequence number, it immediately raises its own next sequence number to match.
* The unique ID is used to break sequence number ties.


* **Vector clocks** are an alternative more heavyweight algorithm.


* Systems using these schemes are eventually consistent across all nodes.
* However, this is insufficient if there are requests (like e.g. asking for a unique username) that need to be aware of every previous operation at runtime.
* Supporting such operations requires knowing, eventually, the total order of operations. The systems that provide this are performing **total order broadcast** (also known as **atomic broadcast**).
* Total order broadcast requires two operational capacities:
  * Reliable delivery (no message loss)
  * Totally ordered delivery (messages are delivered to all nodes in the same order).
* Implementing consensus requires implementing total order broadcast. This feature is provided by e.g. etcd and Zookeeper.


* Linearizable systems can be used to build total order broadcast systems, and vice versa.
* While these two properties are different, having one allows you to have the other, if you desire.



## Consensus
* **Consensus algorithms** are a class of algorithms for addressing the node consensus problem.
* No matter how you design a distributed system, there will always be situations where all nodes must agree on state. For example, transactions across data partitions on different nodes, or leader elections.
* Total order broadcast is an identity transform of a consensus algorithm.


* The classic algorithm for this is the **two-phase commit**.
* Do not confuse this with two-phase locking!
* Two-phase commit relies on a **coordinator**, usually a separate process, which performs a pre-flight check on all of the nodes involving, asking if they can perform the op. The nodes check and reply yes or no. If any nodes say no, the state change is aborted. If all nodes say yes, the coordinator sends a green light.
* This algorithm obviously relies on atomic broadcast to work.
* This algorithm has one glaring weakness: if the coordinator goes down after all nodes ack but before it can send a green or red light, it's non-obvious how to recover (restarting the coordinator, sure, but that takes time).
* If nodes are blocked during this period (they probably are), this is really bad, as it stops the system dead.


* Two-phase commit is a simple idea, but it turns out to make for a crappy consensus algorithm, due to this exact problem.
* Good consensus algorithms avoid this problem. The field includes Paxos (the oldest), Zab, Raft, and VSR.
* The consensus algorithms are all designed around epochs and strict quoroms.
* Each time a leader loss occurs, a quorom of nodes is gathered to vote on a new leader. The new leader increments the epoch number.
* Every time a leader wants to perform an action, it checks whether or not another leader with a higher epoch number exists.
* To do this, it asks a quorom of nodes what the highest epoch number they have seen is. The insight: within a strict quorom, at least one of the nodes, at a minimum, was present in the most recent vote!
* Thus if a leader does not learn from its quorom of another higher-epoch leader, it knows it can safely perform the op.


* Consensus algorithms implement synchronous replication. Totally ordered atomic broadcast, we thus learn, requires we be synchronous.
* Therefore it is very slow, particularly if the network is bad.
* Additionally, certain network partitions can lead to very bad worst-case behavior, such as continuous elections.
* Designing consensus algorithms that are more robust to network failures is an ongoing area of research.


* The chapter ends with a discussion of the sorts of things ZooKeeper is used for.
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
# Chapter 10 --- Batch Processing
## MapReduce

* We can place the nearness data processing systems on a continuum, between online systems on one end and **batch processing systems** on the other end (with stream processing as an intermediate; another chapter).
* Batch processing systems process data on a scheduled or as-needed basis, instead of immediate basis of an online systme.
* Thus the concerns are very different. Latency doesn't matter. We design for total application success or total failure. **Throughput** is the most important measurement.
* Batch processing is really the original programming use case, harkening back to the US Census counting card machine days!


* Discussion of the UNIX philosophy omitted; I know it well.


* Discussion of the basic MapReduce architecture omitted; I read the paper.


* A significant issue in MapReduce is skew. If keys are partitioned amongst reducers naively, hot keys, as typical in a Zipf-distributed system (e.g. celebrities on Twitter), will result in very bad tail-reducer performance.
* Some on-top-of-Hadoop systems, like Apache Pig, provide a skew join facility for working with such keys.
* These keys get randomly sub-partitioned amongst the reducers to distribute the load.


* The default utilization is to perform joins on the reducer side. It's also possible to perform a mapper-side join.


* You do not want to push the final producer of a MapReduce job to a database via insert operations as this is slow. It's better to just build a new database in place. A number of databases designed with batch processing in mind provide this feature (see e.g. LevelDB).

##  Dataflow

* MapReduce is becoming ever more historical.
* There are better processing models, that solve problems that were discovered with MapReduce.
* One big problem with MapReduce has to do with its **state materialization**. In a chained MapReduce, in between every map-reduce there is a write to disc.
* Another one, each further step in the chain must wait for *all* of the previous job to finish before it can start its work.
* Another problem is that mappers are often redundant, and could be omitted entirely.


* The new approach is known as **dataflow engines**. Spark etcetera.
* Dataflow engines build graphs over the entire data workflow, so they contain all of the state (instead of just parts of it, as in MapReduce).
* They generalize the map-reduce steps to generic **operators**. Map-reducers are a subset of operators.
* The fact that the dataflow engines are aware of all steps allows them to optimize movement between steps in ways that were difficult to impossible with MapReduce.
* For example, if the data is small they may avoid writing to disc at all.
* Sorting is now optional, not mandatory. Steps that don't need to sort can omit that operation entirely, saving a ton of time.
* Since operators are generic, you can often combine what used to be several map-reduce steps into one operation. This saves on all of the file writes, all of the sorts, and all of the overhead in between those moves.
* It also allows you to more easily express a broad range of computational ideas. E.g. to perform some of the developer experience optimizations that the API layers that were built on top of Hadoop performed.


* On the other hand, since there may not be intermediate materialized state to back up on, in order to retain fault tolerance dataflow introduces the requirement that computations be deterministic.
* In practice, there are a lot of sneaky ways in which non-determinism may sneak into your processing.


## Graph processing

* What about graph processing, e.g. processing data using graph algorithms?
* Dataflow engines have implemented this feature using the **bulk sychronous parallel** model of computation. This was populared by Google Pregel (Google again...).
* The insight is that most of these algorithms can be implemented by processing one node at a time and "walking" the graph.
* This algorithm archetype is known as a **transitive closure**.
* In BSP nodes are processed in stages. At each stage you process the nodes that match some condition, and evaluate what nodes to step to next.
* When you run out of node to jump to you stop.
* It is possible to parallelize this algorithm across multiple partitions. Ideally you want to partition on the neighborhoods you are going to be within during the walks, but this is hard to do, so most schemes just partition the graph arbitrarily.
* This creates unavoidable message overhead, when nodes of interest are on different machines.
* Ongoing area of research.


## Declarative query languages
* There has been a move to a SQL-like declarative query languages with dataflow engines.
* Also, a whole bunch of useful algorithms are "baked in" the ecosystem. E.g. there are machine learning specific Spark libraries!
# Chapter 11 --- Streams

* Stream processing is a near-realtime, high-throughput alternative to batch processing.
* The smallest element of a stream is an **event**: a small, self-contained, immutable object representing some chunk of state.
* An event is generated by one **producer**, but processed by multiple **consumers**.
* Related events grouped together form a **topic** or **stream**.
* Streams follow a **publish/subscribe model**.


## Messaging systems
* At the core of a stream is a messaging system. The messaging system is responsible for receipt and delivery of events.
* Broadly speaking, stream processing systems can be differenciated by asking two questions about their top-level functionality:
  * What happens when the producer publishing outstrips consumer processing? There are three approaches: queueing, dropping messages, and **backpressure** (the latter being blocking the producer until the current set of messages have been processed).
  * What are the durability guarantees? Messaging systems that may lose data will be faster than ones that provide strong durability.
* You can use any of the tools discussed in the data flow section to implement a message system:
  * Direct producer-consumer messaging.
    * This is simple, and doesn't require standing up new services, but mixes concerns (and doesn't scale very well because of that) and complicates application logic.
  * Message brokers.
    * This makes the most sense at scale.


* When multiple consumers are subscribed to a topic, two patterns are used.
  * **Load balancing**. Each message is delivered to one consumer.
  * **Fan-out**. Each message is delivered to all consumers.


* The same consistency and isolation concerns that were present in databases emerge with message streams as well.
* Message brokers that guarantee delivery wait until an ack response from the delivery target before they dequeue a message.
* If no ack comes within a certain timeout, and a load balancing pattern is used, the message is re-delivered to a different node.
* This means that messages do not necessarily arrive in order!
* Furthermore, the message might have actually been processed, but the ack was dropped or delayed by a network problem.
* So duplication of data between consumers is also possible.

 
## Log-based message brokers
* Message brokers may keep everything in memory, and delete messages that have been sufficiently acked.
* Or they may choose to persist their contents to a local log. Message brokers that choose to do so are **log-based message brokers**.
* Log-based message brokers have the advantage of persistance, whilst still being able to reach extremely high throughputs.
* They are also naturally sequential.
* The throughput comes via partitioning. Multiple brokers may be deployed, each one responsible for a single topic.
* For consumers, reading a message is as simple as reading from the (append-only) log file.
* Fan-out is trivial to implement in this case.
* The load balancing pattern can be implemented by assigning nodes on a per-partition basis. However, this is admittedly much less flexible than is achievable with a non log-based message broker, as it requires having at least as many brokers (and topics) as you have consumers.
* In this mode, consumers maintain an offset in the log file. The broker may periodically check which consumer has the furthest-back offset, and "clean out" or perhaps archive the segment of the log that predates it.
* This greatly reduces network volume, as acks are no longer necessary. This helps make log-based message brokers more easily scalable.


## Syncing aross systems

* Cross-system sychronization is an often necessary task, but it's one fraught with danger.
* The simplest solution would be to perform a full table dump, from time to time, and have the other system parse that dump.
* If this is too slow, or has too much latency, you can use **dual writes**: have the application explicitly write to both systems. This is a bundle of fun of race conditions and failure modes, however. You cannot do it fully safely without cross-system atomic commit, which is a big task to consider.


* An increasingly popular alternative is **change data capture**.
* Databases maintain a log of actions in either their log (log and SSLTable -based architectures) or their write-ahead log (B-tree architectures). Another application could consume that log.
* This is better than dual writes in many cases because, as with log-based message brokers, you are explicitly sychronized.


* Some operations require the full database state to work. For example, building a full-text search index.
* The replication log by default is a database-specific implementation detail. Particularly write-ahead logs will get cleaned up from time to time.
* To continue to have "all the state" without having to maintain an infinitely long log, you can do one of two things:
  * Pair snapshots of the database at a certain timestamp with the replication log thereafter. This allows you to delete log entries accounted for in the most recent snapshot.
  * Perform log compaction on the CDC side. This preserves only the most recent value set, which is what you need.
* Message brokers that support these features (Kafka does) effectively implement durable storage. In other words, you could use this mechanism to implement a Kafka -based database!


## Event sourcing
* **Event sourcing** is a service design pattern (one of Martin Fowler's three event-driven design patterns; see his GOTO talk).
* It is a way of organizing entire systems using events.
* A database allows destructive operations: overwrites and deletes.
* Immutability has big advantages and disadvantages.
* You can capture the current state of the database using the event log and/or a database snapshot.
* But now you have a full history of changes, including changes that have since been overwritten.
* This makes it easier to e.g. debug your system, as you can just point to the place in history where a failure occurred and step through it.
* On the other hand truly deleting data (as necessary for e.g. legal compliance) becomes a problem.


## Time

* Time is a problem.
* When do you know that you have every event within a certain time window? You don't, there could be another event communication stuck in network traffic!
* You have to do something to deal with **straggler events**. You could either:
  * Ignore them, past a certain maximum wait time.
  * Issue a correction to downstream systems relying on event timing.


* Another problem is with wall clock times. Remember, clocks can be out-of-sync.
* You can't necessarily trust the clock time of any device owned by an end user, due to the possibility of subversion.
* So most streaming systems just lop a ton of different timestamps (server recieve time, server send time, end-user read time) together and deal with the combination of attributes these expose.


* Joins between streams are time-dependent, because streams in general do not gaurantee the order of their contents.
* The possibility of a stream join mutating do to event order changes is known as a **slowly changing dimension** in the data warehousing literature. The solution generally used is timestamping each version of the join, so joins are considered non-deterministic!


## Fault tolerance
* How do you tolerate faults in a stream-based system? The answer is not immediately obvious.
* We cannot simply retry the operation, as we had with batch processing, because a stream never ends.
* How do we still provide **exactly once execution** with streams?
* One method is to organize stream events in **microbatches**. A microbatch is complete when all necessary acks are recieved from the subscribers to a selected batch of a topic.
* For example, Spark Streaming is organized around microbatches that are a single second in length.
* Apache Flink dynamically chooses batch sizes and boundary conditions.
* Another approach is to implement an atomic commit protocol.
* Another is to rely on (or force) idempotent operations. E.g. only allow operations that, when applied multiple times, work the same as if they were applied once.
* If your stream processors are idempotent fault tolerance becomes *much* less of an issue!
