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
