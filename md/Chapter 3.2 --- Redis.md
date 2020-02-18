# Redis

## Mission statement
* Redis is a free and open source in-memory key-value store.
* It is like a more advanced version of memcached, the subject of the previous section.
* Redis stands for "REmote DIctionary Server".
* It was originated by a guy at VMWare, and has since spun off twice to a dedicated maintainer in Redis Labs.
* Like memcache it is designed to be blazing fast, and most often used as a cache layer.
* Persistence to disc is configurable via either writing to a log or by dumping to disc (snapshotting) at regular intervals.
* Thus Redis is great for blazing-fast, *mostly* consistent storage.
* You can use it as a cache by disabling the persistance layer entirely (gotta go fast).

## Data model
* The Redis data model is very similar to the memcached one.
* cluster config?
* A word in Redis is a string. Interesting choice. Redis supports a variety of structures involving strings: hashes, lists, sets, sorted sets, bitmaps, hyperloglogs, and geospatials (via geohashes).


* The cluster is managed as a sequence of masters and slaves.
* When the network is fully operational, masters asynchronously send key-store modification traffic to the slaves, which replicate the data locally themselves.
* If a network partition occurs (a timeout or something else), in order to heal, the slaves will ask the master for first a "partial synchronization", where all of the missed updates are batched in one send, and then if that is not possible (due to a high volume of updates not recieved) request a "full synchronization", which requires a much slower backup-and-push on the past of the master.
* This synchronization configuration has high performance, but also high latency, fitting the Redis philosophy perfectly.
* You can optionally request synchronous replication to a specific number of nodes. However in the case of a failover, in some cases it's still possible to lose that update. Redis is not for persistance!
* A master can have multiple slaves.
* Slaves can be connected to one another. This will traffic data replications against one another in exactly the same manner as the master-slave traffic.
* Replication is non-blocking on the master side, except in the case of a failure requiring full synchronization to recover from.
* It is lightly blocking on the slave side: by default slaves will serve using the old copy of the dataset, but there is a brief period when the slave must switch to the new dataset version (delete old data, insert new data) where it blocks.
* Why replicate at all? Reliability, for one thing, but additionally because slaves can be used to farm off long-running requests.
* Since slaves are full dataset replicas, this architecture is obviously insufficient for large data volumes, as it results in a lot of duplicate data!
* High-availability and automatic clustering are available via a few different feature staple-ons.
