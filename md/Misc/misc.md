# TrailDB

## Mission statement
* TrailDB is a database for (1) exploring time-series data (2) with locality (3) optimized for analytics.
* First, time-series data. The TrailDB data model has good performance on time-based tasks, like computing window functions or period aggregations.
* Second, locality. A time-series database is optimized for working with time-series data. However, a regular time-series database does not impose any logical structure on or otherwise keep events from the same emitter close together. TrailDB does, which makes it faster and more workmanlike for such queries.
* Finally, it is optimized for analytics. The database components are supposed to be simple and portable, at the cost of feature surface.

## Data model
* Database is a single file, as in SQLite.
* Once created, the file is read-only. Thus you can use this architecture for analytics but not as a production service!
* Thus TrailDB strongly asserts what is known as referential transparency.
* Data is organized along two axes, users (I will call them emiters here, as this idea is more general than users) and trails.
* The architecture is a key-value store with emitter IDs (user IDs) as the keys (keys are 128-bit values) and the trails as the values.
* Trail consist of sequences of timestamped events. They are organizing in ascending time order.
* One event contains multiple items. Events consist of a field ID and a value ID. The first field ID is always `NULL`. Items are encoded on disc as a 64-bit integer.
* Thus everything is represented on disk as an integer. Notice that this means that the data is highly efficiently stored (everything is 64-bit integers on disc, plus some metadata), but limits the maximum cardinality of your data.
* Going further, everything is represented in a compressed way. The manner in which the data is compressed is moderately complex. It takes advantage of regularities in the data, and is done in blocks of stuff and not on everything all together (in contrast to GZip), so that you only decompress what you need.

## Operations
* Looking up a trail by ID is $O(log N)$. This is a win over time-series databases, which can only return individual timestamped events at this speed.
* Iteration over the keys is also fast.
* Data is stored in integer format, and parsing into e.g. string fields can be done via included utilities. Most databases have a similar store-by-ID implementation, but do not expose this fact anywhere. TrailDB chooses to expose you to this because they believe you can leverage som speed advantage from it, if you pick IDs intelligently.
* You can build the database and you can merge two databases from unordered streams of data. Merging databases is an efficient operation.
* Maping a string to an item is $O(L)$, where $L$ is the number of possible values that item can take on.
* Cursors are cheap.
* TrailDB implements virtual views as "event filters". You can specify at query time that only records matching certain conditions be returned. Event filters expedite sub-selection operations.
* You can also get materialized views, e.g. export event filters to separate TrailDB files.
* You can define a multi-cursor that connects to multiple TrailDB instances. This is useful because a common 
# JanusGraph

## Mission statement

* JanusGraph is an open-source, Linux Foundation supported graph database.
* It's meant to be a completely open source alternative to the more popular Neo4J, which is corporate copyleft.
* It's designed to scale horizontally across crazy numbers of compute nodes.
* It supports Hadoop job organization and graph queries using the Gremlin declarative structured query language (which has an Apache-developed Python binding, in addition to a as-you-would-expect Java binding).
* It does not come with a backing store baked-in; you need to select one yourself.
* The backing store in production may be either Apache Cassandra, which is continuously available but inconsistent, or HBase, which is strongly consistent but doesn't have high availability guarantees (cf. CAP theorem).
* An Oracle Berkeley DB Java Edition (BerkeleyDB JE). This should only be used for testing and exploration purposes however.
* Indices may also be used. These will speed up and/or enable more complex queries. The options are Elasticsearch, Solr, and Lucerne.
* You can use it either monolithically (queries are executed within the same JVM) or as a service.
* JanusGraph seems to have a so-so but not too-large userbase.
* It seems to be backed by IBM, Google, and the Linux Foundation.
* AWS is the only hosted graph database provider as far as I can tell. They provide AWS Neptune, which is...a closed source unknown. OK.

## Configuration
* A JanusGraph graph database cluster consists of one or more JanusGraph **instances**.
* Instances are built using a **configuration**.


* A console is included in JanusGraph. This can be used to e.g. open graphs and inspect them using the built-in Gremlin tooling.
* The configuration of the graph and the location of the graph files are specified by a file, which you can at run-time by whatever you use to initialize JanusGraph. From the console this looks like so:

        graph = JanusGraphFactory.open('path/to/configuration.properties')

* Dealing with configuration details gets complicated quickly. A temporary-most configuration is:

        storage.backend=berkeleyje
        storage.directory=/tmp/graph


* At a minimum, a storage backend must be specified. Optionally, a indexing backend and some cache configuration may be provided, if performance is a concern.


* On its own JanusGraph is just a bunch of JAR files. It may be used on the same thread as a program running on the JVM, or it may be ran as a persistent server process.
* Doing the latter means deploying a process known as the JanusGraph Server. JanusGraph Server uses the Gremlin Server as its service layer.
* Unlike the storage configuration, this configuration is specified (1) prior to execution and (2) at a specific file location, `./conf/gremlin-server` (with respect to the JanusGraph folder). A simple server configuration is:

        graphs: {
          graph: conf/janusgraph-berkeleyje.properties
        }
        plugins:
          - janusgraph.imports


* Graphs bound to the server point to specific configuration files, and the server proceeds to use that configuration file to set itself up.
* To actually int, run `bin/janusgraph.sh start`.


* The configuration layer allows applying configurations in five different scopes (roughly speaking, from local to a single instance to global for the entire cluster).
* The configuration of the first instance is inherited from the configuration file.
* The configuration of subsequent instances may be modified by calling on certain functions in the configuration layer of JanusGraph.
* Old instances do not inherit changes to the configuration details (e.g. there is no built-in rolling update support).
* To update configuration details for an entire cluster, you need to manually pause the process: close all but one instance, ensure that said instance is offline and done processing transactions, apply the update, and rescale back up.
* This is of course very clunky. In practice you're probably better off scaling with changes at the cluster management layer.


* The recommend server gateway is a WebSocket connection. A custom subprotocol is used for communications with JanusGraph, which again comes from Gremlin.
* The quickest way to test for a connection is with Gremlin Console. You can `:remote connect tinkerpop.server conf/remote.yaml`. In this command, `tinkerpop.server` is the Gremlin plug-in that enables server connecting, and `conf/remote.yaml` is a pointer to the file (it doesn't have to be this specific name) that provides configuration details for Gremlin's connection.
* It is also possible to configuration communication via an HTTP API. However, this requires a lot of additional work, as far as I can tell, so I'm not sure why you would want to do that.

## Data model

* JanusGraph uses a graph theory -style vertex-edge schema (as opposed to a semantic web -style triple schema).
* It can be either schema-on-read or schema-on-write. Schema-on-write, e.g. specifying and following an explicit schema, is heavily encouraged.
* You can (should?) turn off implicit schemaby specifying `schema.default=none` in the cluster configuration file.
* All schema details are handled via the management layer (the same one that controls instance configuration). So, if you go the explicit schema route, you want to define an explicit schema before creating any instances.


* An important schema configuration detail is **edge label multiplicity**. Edges between vertices may be one-to-one (simple), one-to-many, many-to-one, or many-to-many.
* Graph databases always handle directed graphs.
* One-to-one and many-to-many relationships are easy to understand.
* A many-to-one relation specifies that there may be at most one outgoing edge but unlimited incoming edges. A `IS CHILD OF` relation is a good example (a single person has one parent, but one parent can have many children).
* A one-to-many relation is the opposite. A `PARENT OF` relation, which is the inverse of the relation above, is a good example of such.


* Both vertices and edges have a label and a key-value store of properties.
* Edge labels and property keys are jointly known as relation types, and must be uniqueley named. So you cannot have both an edge and a property with the same name.
* Properties on verticles and edges are key-value pairs having a certain type.
* The list of types is about what you would expect from a database. There's no blob, but there is a geoshape.
* The values your keys map to may be singletons, lists, or sets. This is **property key cardinality**, and can be configured at schema definition time as well, obviously.
* Edge labels are the only requirement, all of vertex labels, vertex properties, and edge properties are optional.
* The list of valid vertex labels is another schema configuration detail that can be specified at start time.


* Schema changes that do not touch existing data propogate safely. Not sure if there's a way to tell if it's hit all of your instances yet.
* Schema changes that touch existing data (e.g. database migrations) will fail. You need to perform a database migration, which in graph theory land means a batch graph transformation, for those to go through. That can only be done offline.
* The one exception appears to be edge label changes. Edge label updates can happen online, but will not touch existing vertices or edges.
* You an also add indexes (next section) on the fly.


* Ghost vertices are a problem on eventually consistent backends. Sounds a lot like something I read could happen in MongoDB:

    > When the same vertex is concurrently removed in one transaction and modified in another, both transactions will successfully commit on eventually consistent storage backends and the vertex will still exist with only the modified properties or edges. This is referred to as a ghost vertex. It is possible to guard against ghost vertices on eventually consistent backends using key uniqueness but this is prohibitively expensive in most cases. A more scalable approach is to allow ghost vertices temporarily and clearing them out in regular time intervals.


## Indexing

* Graph databases, like regular transactional databases, can get significant speed boosts from using indices.


* Most graph queries start by looking at a list of vertices or edges, and you can speed these queries up by keeping a sorted list of such indices in memory.
* JanusGraph supports two index types: **composite** and **mixed**.
* Composite indices are multi-column B-tree indices. They are good for searching arbitrarily deep into an ordered list of attributes. For example, if you have an index built on $(A, B, C)$, where $A$ is the first indexed property and $C$ is the last. You can index $(A)$, $(A, B)$, $(A, B, C)$, but it'll be no use for $(B)$ or $(B, C)$ queries.
* Mixed indexes are just multiple single-column indices, best I can tell.
* Composite indices are built-in. Mixed indices require shelling out to an indexing backend like ElasticSearch.
* You can explicitly disable full graph scans with the `force-index` configuration option. This is recommended for large graphs at production scale.
* The details of how these indexes work, and thus what they can and can't do, are at the mercy of the index store you are using. That's a subject for another set of notes...



* Graph databases (and JanusGraph) can also have **vertex indices**. These are indices on the vertices of the graph. They are useful when performing vertex-centric queries, e.g. queries that have reason to scan all or a significant subset of the edges of specific vertices. Vertex indices speed up searching through these relationships.
* This is a useful property to have for graphs where vertices may have 100s or 1000s of edges.


* You can await the completion of an index creation in flight in a synchronous way using an `awaitGraphIndexStatus` hook in the management layer.

## Transactions

* JanusGraph is only ACID on the BerkeleyDB JE backend. Neither Cassandra nor HBase, the two backend options, provide this guarantee.
* **Transactional scope** is the behavior of pointers with respect to the database transaction.
* JanusGraph is obvious transactional. When defining a new vertex of edge, we may assign a reference to that vertex of edge to a variable in the language we are performing the operation in.
* Once the change is committed, if the variable pointer remains valid we state that it has "transitioned transactional scope". If it is no longer valid, that variable is lost because it has "left transactional scope".
* This is a non-trivial feature because the object reference has to reflect the value currently in the database. There may be a difference between commit time and update time (especially in an explicitly "eventually consistent" database), which can cause the object data and the database data to desync.
* JanusGraph will transition vertex references for you, but it will not transition edges.
* If you want to transition edges you have to do so manually.


* JanusGraph includes some logic for attempting to mitigate temporary transaction failures due to things like e.g. network hiccups. Eventually it will give up (configurable after how many retries).
* Permanent failures, like lock failures due to race conditions, may also occur. In these cases you will get a transaction error, which you will need to handle somehow.


* JanusGraph supports multi-threaded transactions. You can isolate a transaction to a specific thread. This can be used to speed up many (embarrassingly parallel) graph algorithms.
* You can also use thread isolation to avoid potential lock contention on long-running transactions where at least one of the operations has to lock the database.
* Wrap that sub-transaction into a separate thread, and commit that in the body of the transaction. This will force a (sort of a synchronous) lock and make the rest of the transaction respect it (still not 100% clear on this point; see 10.6 in https://docs.janusgraph.org/latest/tx.html).


## Caching

* JanusGraph offers multiple levels of caches. From closest to furthest they are transaction-level caches, database-level caches, and storage backend caches.
* The transaction level cache stores the (1) properties and (2) adjacency list of accessed vertices, and access locations in any indices, in memory.
* This cache is local to each transaction, e.g. it gets flushed at the end of the transaction.
* This cache is most useful for queries that hit the same or similar nodes many times, obviously. It's something to take advantage when designing transactions.
* The next layer of cache is the database-level cache. This cache retains data across multiple transactions, and only covers adjacency list access hotspots. The database-level cache can significantly speed up read-heavy workloads.
* Finally, the storage backend providers maintain (usually) their own data caching layer. This is typically more compressed, compacted, and much larger than the in-process one. It also manages memory outside of the JVM typically (it is "off-heap") so it is less prone to slowdown. But it is the slowest of the cache layers too, for the same reason.

## Logging

* JanusGraph includes a user transaction logging feature.
* This can be usedas a record of change (e.g. for log parsing), for downstream updates in a more complex system, and for defining triggers on certain events.
* These are useful because using the logs for this purpose, instead of querying the database directly, doesn't slow the process down.
Look at:
* http://sql2gremlin.com/
* https://docs.janusgraph.org/latest/getting-started.html
* Get the JanusGraph server process set up. Then run http://tinkerpop.apache.org/docs/current/reference/#gremlin-python against it.
## MapReduce

* MapReduce is a programming model for compute jobs.
* A **programming model** is a specification for (and/or implementation of) a paradigm that involves concepts distinct from those in the language calling it. 
* E.g. MapReduce is conceptually distinct from the C programming language, or Scala, and so on. This distinguishes it from a simple library!
* Compute jobs that are organized using the MapReduce framework are automatically parallelizable in way that is optimized for large-scale, distributed systems.
* And it is flexible. You can apply MapReduce to a vast range of compute jobs.
* It was introduced by a Jeffrey Dean paper years ago, and was the primary programming model for large-scale compute jobs at Google for a long time.
* In the open source community it is implemented via Apache Hadoop and the related ecosystem of tools.
* More recent work is pushing past the boundaries imposed by MapReduce, however!


* Large data jobs tends to involve simple computations that get performed against extremely large amounts of data.
* When the data volume is large enough that a single machine cannot easily handle it, many machines are used for processsing.
* With an a la carte system, the end result is a very small amount of simple computational code, and a very large amount of boilerplate dealing with parallelization, fault-tolerance, data distribution, and load balancing.
* The MapReduce system hides these messy details from you behind a simple abstraction.


* Each MapReduce job consists of a map and a reduce.
* In the **map** step, individual records (modeled as key-value pairs) are processed and turned into arbitrarily many intermediate key-value pairs.
* In the **reduce** step, pairs with the same key are co-processed into some final result.
* In both the map and reduce steps, the operations are atomic to individual data structures and are presented via an iterator. This is what makes this tractable memory-wise!
* For example, you could implement a simple word counter by emitting key-value pairs of the form $\{word \: : \: 1\}$, then writing a reduce function that sums across these key-value pairs (resulting a word-by-word sum of 1s, or a count!).
* In reality, the reduce step requires data with the same key to be colocated. Thus there is actually a third step: shuffle, which moves the intermediate mapped representations with the same key to the same memory space or node.
* Thus it might be more accurate to say MapShuffleReduce.


* The operational implementation described in the paper is as follows:
1. The size of the data is determined and divided by user-specified memory partition sizes. A mapper split size $M$ is calculated, along with a reducer split size $R$.
2. That many separate MapReduce processes are spun up on the cluster or machine. One process is the master, and it assigns work to the rest of the processes.
3. Map workers that are told by the master what data to work on parse the requisite key-value pairs out of the data, and start running the map operation on them.
4. The map workers hold the outputs in memory, but periodically flush them to disc in $R$ partitions (partitioning is handled by the partitioner). The location of these partitions is sent to the master.
5. The master assigns the partitions as they come in to reducers. The reducer reads that data in and sorts it.
6. The reducer applies the reduction operation to the data. The result of the reduction is appended to an output file when the reducer is done operating.
8. Once all of the reducers have finished running, the MapReduce system wakes up the overlying user program.


* Fault tolerance is provided via health polling from the master. If the master does not recieve a response from a polled worker in enough time, it marks it failed and moves that job onto another worker.
* Importantly, work that was already performed (or will be performed) by the failing worker is replicated by the reassigned worker. This is because the written-to resource is a local disc inaccessible to the master.
* The additional complexity of making the disc accessible might not be worth it!
* Reducers that were in the process of reading from the old map worker stop, and begin listening for input from the new map worker instead.
* Master failures are considered unlikely, and there is no provisioning for them in the original MapReduce paper. However, a periodic checkpointing scheme is described as being easy-to-implement.
* Hadoop implements safeties for this case.
* Note that if a worker node failure causes a reassignment, and the worker node eventually completes its job and signals the master about this, that signal is ignored.


* Locality is achieved by the placement of map and reduce processes near to data being mapped or reduced.
* This seems to be a system-specific implementation detail.


* Unplanned machine resource limitations are dealt with via back-up mechanism.
* The problem is that due to a host of reasons, but most often resource contention, machines may run code slower than expected.
* These stragglers will slow down overall execution time.
* The Google MapReduce system solves this by brute-forcing it, pretty much. When the job is close to completion, the remaining tasks are spun up on new machines.
* Now the first machine that completes the instruction (whether the original or the new backup) sets the standard. The job from the second machine is ignored.
* You could obviously increase the concurrency and/or back up at an earlier save point to further increase the resiliance and time-performance of the system, albeit at great computational cost.


* Some optimizations follow.
* You could specify a better partitioning function. The default is a hash-based one, but if you would like to have data grouped by, say, URL host, then a custom partitioning function may be specified and used.
* You may optionally gaurantee that the map and reduce jobs process and emit data in sort order.
* Transfering the data from the mapper process to the reducer process involves a network trip. To reduce the volume of network travel, you may optionally specify a **combiner**.
* A combiner partially reduces the response (for example, by throwing out non-interesting keys or creating an `OTHER` category or something, a need I am very familiar with!). It runs on the same machine as the map process, and is run on the map result *before* emission to the reducer.
* This can significantly reduce network resource utilization.
* The implementation describes assumes inputs are key-value pairs, but you can implement a seperable **reader** for any other input data type.
* A sequentially executing MapReduce implementation is a good thing to have, as it allows you to more easily debug otherwise asynchronous jobs.


* Hadoop is the main open source implementation of MapReduce, and what's used in production really.
* However, MapReduce is pretty old now. It's getting replaced by Spark and 
## Mission statement

* HBase is a wide-column database (for more on what these are look at the Cassandra notes).
* However, whilst Cassandra emphasizes availability, HBase emphasizes consistency.
* HBase is based on HDFS as the underlying data store. HDFS is designed to be highly durable.
* HBase provides good high-volume, large-data, random read-write performance.
* Its primary use case is "reporting".


## Zookeeper
* HBase makes use of another Apache project, Zookeeper.
* Zookeeper is a high availability, strongly consistent, totally ordered log-based key-value store.
* It stores the configuration details for HBase in a durable way, and is meant to be a reusable component that can be plugged into other database architectures.
* Zookeeper is designed to address a need common to every database implementation, which is providing configuration details in a durabile way.


## Data model and architecture
* The HBase server is sharded into a master service and slave daemons.
* All of the coordination services live on the master. All of the data lives on the slaves.
* There are many services in play; the architecture is relatively complex.
* HBase has the concept of a column family. A column family is a set of commonly group-accessed columns which are grouped together in memory. This improves read performance.
* HBase also supports having multiple versions of a dataset entry.


* Writes are slow (why?).
* Reads are fast! HBase is the data store of choice for large Hadoop jobs. Hadoop is the job framework specifically designed for running jobs against data at scale!
# Red-black trees

* Red-black trees are a type of self-balancing binary search tree. They have $O(\log{n})$ read times, making them highly efficient.
* In addition to the properties of a binary search tree red-black trees have the following invariants:
  * Each node is either red or black.
  * All leaves are `null` and black.
  * If a node is red, then both its children are black.
  * Every path from a given node to any of its descendant `null` nodes contains the same number of black nodes.
* Red–black trees offer worst-case guarantees for insertion time, deletion time, and search time.
* Read operations are the same as in binary search trees.
* Insert and delete are much more compliated, but still $O(\log{n})$.
* Here's the tutorial I'm following: http://staff.ustc.edu.cn/~csli/graduate/algorithms/book6/chap14.htm.


* A


```python
class Node:
    def __init__(self, color, left, right, parent, value=None):
        self.color = color
        self.left = left
        self.right = right
        self.parent = parent
        self.value = value
        

class Tree:
    def __init__(self, root):
        self.root = root
        
        
def set_left(node, parent):
    # If the node is the right child of the parent, remove it from there.
    if parent.right == node:
        parent.right = None
    
    # If the parent has an existing left child set that child's parent to None (overwrite).
    if parent.left is not None:
        parent.left.parent = None
    
    # Attach the node to the parent.
    parent.left = node
    
    # If the node is non-null assign it the parent pointer.
    if node is not None:
        node.parent = parent
    
        
def set_right(node, parent):
    if parent.left == node:
        parent.left = None
    
    if parent.right is not None:
        parent.right.parent = None    
    
    parent.right = node
    
    if node is not None:
        node.parent = parent

    
def left_rotate(T, x):
    y = x.right
    
    set_right(node=y.left, parent=x)
    
    if x.parent is None:
        T.root = y  # TODO: ?
    else:
        if x == x.parent.left:
            set_left(node=y, parent=x.parent)
        else:
            set_right(node=y, parent=x.parent)
            
    set_left(node=x, parent=y)
    
    
def right_rotate(T, x):
    y = x.left
    
    set_left(node=y.right, parent=x)
    
    if x.parent is None:
        T.root = y
    else:
        if x == x.parent.right:
            set_right(node=y, parent=x.parent)
        else:
            set_left(node=y, parent=x.parent)
            
    set_right(node=x, parent=y)
    
    
def binary_insert(T, val):
    """Insert as though inserting into a binary tree, with the color red."""
    n = T.root
    while True:
        if val < n.value:
            if n.left is None:
                node = Node('red', None, None, n, value=val)
                set_left(node=node, parent=n)
                return node
            else:
                n = n.left
        else:
            if n.right is None:
                node = Node('red', None, None, n, value=val)
                set_right(node=node, parent=n)
                return node
            else:
                n = n.right
    
    
def insert(T, v):
    # Perform a binary insert.
    x = binary_insert(T, v)
    
    
    # Now correct violations of the red-black property.
    while x != T.root and x.parent.color == 'red':
        if x.parent == x.parent.parent.left:
            y = x.parent.parent.right
            
            # Case 1, both the node and its parent are red. 
            # Simply recolor.
            # This can move the issue further up the tree, which is why we have a while loop.
            # So we can fix those issues as well.
            if y.color == 'red':
                x.parent.color = 'black'
                y.color = 'black'
                x.parent.parent.color = 'red'
                x = x.parent.parent
                
            else:
                # Case 2:
                # * x is not the tree root.
                # * x's parent is red.
                # * x's parent is to the left of its grandparent.
                # * x's uncle (its parent's right sibling) is black.
                # * x is on its parent's right.
                # 
                # x's parent is red but its uncle is black, which violates invariant four.
                # We can fix this by performing a left rotation.                
                if x == x.parent.right:
                    x = x.parent
                    left_rotate(T, x)
                
                # Case 3. After fixing case 2, we will always end up with a doubled red node further up the tree.
                # Perform a right rotation to fix this.
                x.parent.color = 'black'
                x.parent.parent.color = 'red'
                right_rotate(T, x.parent.parent)
                
        else:  # x.parent == x.parent.parent.right
            # Same logic, just with right and left exchanged.
            y = x.parent.parent.left
            
            if y.color == 'red':
                x.parent.color = 'black'
                y.color = 'black'
                x.parent.parent.color = 'red'
                x = x.parent.parent
                
            else:
                if x == x.parent.left:
                    x = x.parent
                    right_rotate(T, x)
                    
                x.parent.color = 'black'
                x.parent.parent.color = 'red'
                left_rotate(T, x.parent.parent)
    
    T.root.color = 'black'
```


```python
# Test suite for the swap operation.

def test_set_left_clean():
    r = Node('red', None, None, None)
    t = Tree(r)
    n = Node('red', None, None, None)
    set_left(n, r)
    assert r.left == n
    assert n.parent == r
    assert r.right == None
    assert n.left == None
    assert n.right == None
    
    
def test_set_right_clean():
    r = Node('black', None, None, None)
    t = Tree(r)
    n = Node('red', None, None, None)
    set_right(n, r)
    assert r.right == n
    assert n.parent == r
    assert r.left == None
    assert n.left == None
    assert n.right == None
    
    
def test_set_left_overwrite():
    r = Node('black', None, None, None)
    t = Tree(r)
    n1 = Node('red', None, None, None)
    n2 = Node('red', None, None, None)
    
    r.left = n1
    n1.parent = r
    
    set_left(n2, r)
    assert r.left == n2
    assert n2.parent == r
    assert n1.parent == None

    
def test_set_right_overwrite():
    r = Node('black', None, None, None)
    t = Tree(r)
    n1 = Node('red', None, None, None)
    n2 = Node('red', None, None, None)
    
    r.right = n1
    n1.parent = r
    
    set_right(n2, r)
    assert r.right == n2
    assert n2.parent == r
    assert n1.parent == None


def test_set_left_swap():
    r = Node('black', None, None, None)
    t = Tree(r)
    n = Node('red', None, None, None)
    
    r.right = n
    n.parent = r
    
    set_left(n, r)
    assert r.left == n
    assert n.parent == r
    assert r.right == None
    
    
def test_set_right_swap():
    r = Node('black', None, None, None)
    t = Tree(r)
    n = Node('red', None, None, None)
    
    r.left = n
    n.parent = r
    
    set_right(n, r)
    assert r.right == n
    assert n.parent == r
    assert r.left == None

    
    
test_set_left_clean()
test_set_right_clean()
test_set_left_overwrite()
test_set_right_overwrite()
test_set_left_swap()
test_set_right_swap()
```


```python
def test_left_rotate_nonroot():
    r = Node('black', None, None, None)
    T = Tree(r)
    x = Node('red', None, None, None)
    y = Node('black', None, None, None)
    
    r.right = x
    x.parent = r
    x.right = y
    y.parent = x

    left_rotate(T, x)
    
    assert T.root == r
    assert T.root.right == y
    assert y.left == x
    
    
def test_left_rotate_root():
    r = x = Node('black', None, None, None)
    T = Tree(r)
    y = Node('black', None, None, None)
    
    r.right = y
    y.parent = r
    
    left_rotate(T, r)
    
    assert T.root == y
    assert T.root.left == r
    
    
def test_right_rotate_nonroot():
    r = Node('black', None, None, None)
    T = Tree(r)
    x = Node('red', None, None, None)
    y = Node('black', None, None, None)
    
    r.left = x
    x.parent = r
    x.left = y
    y.parent = x

    right_rotate(T, x)
    
    assert T.root == r
    assert T.root.left == y
    assert y.right == x
    
    
def test_right_rotate_root():
    r = x = Node('black', None, None, None)
    T = Tree(r)
    y = Node('black', None, None, None)
    
    r.left = y
    y.parent = r
    
    right_rotate(T, r)
    
    assert T.root == y
    assert T.root.right == r
    
    
test_left_rotate_nonroot()
test_left_rotate_root()
test_right_rotate_nonroot()
test_right_rotate_root()
```


```python
def test_binary_insert_left():
    r = Node('black', None, None, None, value=1)
    T = Tree(r)
    binary_insert(T, 0)
    
    assert T.root.left.value == 0
    
    
def test_binary_insert_right():
    r = Node('black', None, None, None, value=1)
    T = Tree(r)
    binary_insert(T, 2)
    
    assert T.root.right.value == 2

    
def test_binary_insert_dig():
    r = Node('black', None, None, None, value=1)
    T = Tree(r)
    binary_insert(T, 0)
    binary_insert(T, 2)
    
    binary_insert(T, 3)
    binary_insert(T, -1)
    
    assert T.root.right.right.value == 3
    assert T.root.left.left.value == -1

    
test_binary_insert_left()
test_binary_insert_right()
test_binary_insert_dig()
```


```python
def test_insert():
    rightright = Node('red', None, None, None, value=15)
    right = Node('black', None, rightright, None, value=14)
    rightright.parent = right
    
    leftleft = Node('black', None, None, None, value=1)
    leftrightright = Node('red', None, None, None, value=8)
    
    # leftrightleftleft = Node('red', None, None, value=4)
    leftrightleft = Node('red', None, None, None, value=5)
    leftright = Node('black', leftrightleft, leftrightright, None, value=7)
    leftrightleft.parent = leftright
    leftrightright.parent = leftright
    
    left = Node('red', leftleft, leftright, None, value=2)
    leftleft.parent = left
    leftright.parent = left
    
    root = Node('black', left, right, None, value=11)
    left.parent = root
    right.parent = root
    
    t = Tree(root)
    
    insert(t, 4)
    
test_insert()
```

* This case above emulates the case given in the reference.
* Not included: deletion, which is also $O(\log{n})$ and "only slightly more complicated than insertion".
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
TODO: http://staff.ustc.edu.cn/~csli/graduate/algorithms/book6/chap14.htm
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
(Speaking notes for a talk on database architecture)

# Part 1
* There are just three properties intrinsic to a database.
  * Persistent insert --- You can push data into the database, and expect it to still be there later.
  * Persistent read --- You can read data out of the database, and expect it to return the same value as later.
  * Delete --- You can delete data from the database.
  
  
* Not included in these properties:
  * Transactions --- transactions are thought of as a common database feature. But they are not an intrinsic thing!
  * SQL --- SQL is jsut a convenient, uniform standard.


# Part 2
* Simplest possible database: append to a file.
* Simple log-structured database.
* Add log merges.
* Why are log files good? Append operation is fast.


* More complicated arch: SSLTables.
* Use a memcache which supports random order, sorted iteration to write log files that are key-sorted.
* Log files are now much faster to merge.
* If a sudden shutdown occurs, the in-memory data is lost. A small write-ahead log is used to recover the state of the memcache in the case of a failure.


* Final architecture: B-trees.
* B-trees sort data into pages, which are accessed from previous pages.
* Also use a write-ahead log.
* To add an item you can split a page.
* Deleting an item is more complicated.
* B-trees involve random seek-writes, but do not necessitate a log merge process.
* So more predictable performance, but worse median write performance.
  

# Part 3
* Example transactional databases
  * SQL databases are not the same, but their user-facing design is the same.
  * SQLite
      * File-based.
      * No concurrency, only one in-flight request at a time.
      * Transactions.
      * Limited SQL implementation.
      * Open-source.
  * Postgres
      * Complicated.
      * Concurrent.
      * Transactions.
      * Broad SQL implementation.
      * Open-source.
  * OracleDB
      * Complicated.
      * Concurrent.
      * Transactions.
      * Broad SQL implementation.
      * Closed-source.


* Classical transactional databases.
  * ACID --- atomic, consistent, isolated, durable.
  * Originally invented for the online transaction processing, or OLTP, use case.
  * That explains a lot about why these properties, particularly the second one, are important.


* External data model
  * Data is modeled using tables.
  * SQL --- Structured Query Language is provided as an abstraction.
      * SQL queries reconstruct data of interest by performing joins between tables.
      * SQL is what is called a "declarative query language". They are designed to look kind of like English sentences.
  * Why do you need SQL at all? So that database writing skill is portable between databases.
  * Why does it need to be declarative? To hide the details of database internals (the internal data model) from you.
  * Why is this useful? Because queries that you write against databases are not immediately executed; instead they are pushed through a query optimizer, which finds ways to speed your query up by using its knowledge of the underlying hardware and software layers.
  * SQL says what you want, not how you want. The alternative is imperative - like writing a program.
  * Unfortunately this abstraction is a bit leaky, due to vendors implementing things as they like them.


* On-disc key-value stores     
    * A key-value store is just a hash maps. Just like in your favorite programming language! 
    * Explain how hash maps work, briefly.    
    * Bitcask is an example. It uses a log-structured file for storage.
    * This is technically NoSQL.
    * But it's a bit _too_ simple to be useful.
    * This arrangement is mainly used for databases used for e.g. persistent configuration services. 
    * For example, Apache Zookeeper.


* In-memory key-value stores
  * If you do not need persistance, you can use an in-memory database.
  * Why? Volatile memory (RAM) is much, much faster than disc.
  * This is what Memcached and Redis do.
  * The memory structures these databases use are also just hash maps.
  * Explain how hash maps work, briefly.
  * The disadvantage is obviously that if the computer shuts down suddenly, everything in the database will be gone.
  * You can make a semi-persistent database by writing to disc occassionally, but the more often you do this the slower your database gets.


* Document store
  * A document store is a kind of more complicated key-value store that treats everything in the database as a document.
  * Documents are basically JSON blobs (though they're not necessarily stored that way on disc).
  * Consider a transaction database. This database is schema-on-write, in that all of the data that you store in it has to be put in a certain format (in terms of rows and columns) when you store it.
  * A document database meanwhile contains documents of any format. You can only know for sure what you have when you parse a document.
  * On the one hand, the idea of a document idea is much more flexible than the idea of a table.
  * On the other hand, it naturally leads to more complex code, as you need to be aware of and catch any document model differences.
  * Use a document database when you often need to access documents "all at once". For this use case they have much better locality than databases, which, when you have to join data, have to look much further.
  * However, do not use it for very complex applications and expect it to work well.
  * MongoDB is an example document store.


* Wide-column stores
  * A wide-column store is a two-dimensional key-value store.
  * It's in between transactional databases and document stores in shape.
  * Meanwhile, while true column stores provide locality on all of the columns, wide-column data stores provide locality on the individual records.
  * Example wide-columns stores are Cassandra and HBase.


* Column-oriented stores
  * Introduce OLAP versus OLTP.
  * OLTP wants to work on tons of columns at a time, not tons of rows at a time. An architecture that provides locality on columns is better.
  * Column-oriented stores basically invert row-based databases.
  * Druid is probably the most common of these.
  
  
* Graph database
  * Good for dealing with interconnectiveness and graph searches.
  * Examples: JanusGraph, Neo4j.


* Stream-oriented datase
  * This is a category I'm inventing for TrailDB.
  * TrailDB is an interesting data model.
  * Explain the data model.
  * This is meant to make it convenient to query user activity or other time-series chained events.


* Point is that there are LOTS of different database models!
