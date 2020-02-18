# Memcached

## Mission statement
* Memcached is a free and open-source in-memory key-value store.
* It was originally developed by and for the LiveJournal website, back in the day.
* It's almost a database, but not quite one. It differs in that it's not persistant. Memcached is allotted a certain amount of memory, and when that memory limit reached, to insert new values memcached starts deleting old values.
* This deletion occurs using the simple "Least Recently Used", or **LRU**, **caching strategy** (in this context this is referred to as the **eviction mode**).
* When you insert data into memcached that you want to access later, you cannot assume that it will still be there.
* The tradeoff is that memcached is ridiculously fast, and has a simple, easy-to-understand architecture.
* Memcached is thus meant to be used as a **caching layer**. Put it in front of your production database to speed up your queries!

## Data model
* Memcached uses a simple hash map log structured storage architecture (this is the simplest practical database architecture; see Chapter 3 notes for more).
* Memcached supports clusters containing multiple nodes. A hash is computed on the data being inserted in order to determine which node the data gets sent to. Then the data is hashed again for storage.
* This is a **shared nothing architecture**. The client knows the locations of the nodes, obviously, but the nodes know nothing about each other, and do not share any resources.
* Memcached is an **in-memory data store**. The nodes are meant to be volatile memory resources.
* There is no type support. A **word** in memcached is a byte.
* As mentioned in the previous section, a least recently used caching strategy is used to purge old data when the service reaches its size limit.

## Security
* memcache emphasized brutal simplicity and efficiency. But in the case of security that simplicity apparently makes it emminently hackable.
* Memcache uses a flat security model, with privileges applying to lots of things all at once. For example, if you have write access, you have all the write access; same with reads.
* When deployed on an unsecured network, it's very easy for external actors to get to, inspect, and even modify a memcached service.
* memcache over UDP is a particular problem. This feature was disabled by default eventually, but for a while you could access cache data for a lot of public websites apparently!
* Since memcache has such lousy security you should implement your own security layer.
