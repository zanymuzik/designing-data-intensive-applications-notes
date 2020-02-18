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
