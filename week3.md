#Week3

##Storage Engine

###Intro

TL;DR : *Interface between Database and the hardware on which the db is running.*

The storage engine handles the interface between the database and the hardware, and by hardware, we mean specifically **memory and disk**, and any representations of the data (or metadata) that is located there, which is to say the **data and indexes**.

####Doesn't do

* Doesn't change the way you perform a query (shell / driver)
* Doesn't change behaviour at the cluster level (no effect on scaling for ex)

####Does affect

* How data is written to disk
* How data is deleted/removed from disk
* How data is read from disk
* Data structures used to store data (still BSON)
* Format of indexes

####Pluggable engines

Two choices :

* **MMAPv1** (original)
* **WiredTiger** (new in MongoDB v3)

###MMAPv1

Uses mmap (memory mapping) system call, lets treat files as if they were already in memory.

* If not in memory : a page fault loads it up in RAM
* If in memory and updated : an fsync propagates changes back to disk
* Default storage engine.
	* or use `mongod --storageEngine mmapv1`

`db.serverStatus()`: look for `storageEngine` entry to retrieve the storageEngine you're running.

####Features

* Collection-level locking (MongoDB >= 3.0)
	* Database-level locking (MongoDB 2.2 - 2.6)
* Journal
	* write-ahead log (insures consistency)
		1. Write what you're about to do to the journal
		2. Then write to the db
		3. Possibilities (consistency is kept)
			* Write to the journal failed : nothing was written in the journal nor in db
			* Write in 	the db failed : you can replay it with the journal when server's goes back up
* Data on disk is BSON
	* Bits are mapped from disk to virtual memory
	
Locks = Shared resources

* Data : multiple readers / single writer lock
	* One writers comes in : locks all readers + any other writers
* Metadata :
	* Indexes : multiple documents sharing an index -> updating one or an other would involve the update of the index causing conflict
	* Journal

###WiredTiger

####Features

* Document-level locking
* Compression
* Lacks some pitfalls of MMAPv1
* Performance gains

* Already used by other DBs
* Open Source

`mongod --storageEngine wiredTiger`

*If launched on data already managed by MMAPv1, will fail*

####Internals

* Stores Data in Btrees
* During update, writes a new version of document instead of overwriting existing one
* Two caches
	* WiredTiger Cache (1/2 of RAM by default)
	* File System Cache
	* How it works : each 60seconds happens a "checkpoint" (each one is a consistent snapshot of the data) :
		* Data goes from Wired Tiger Cache
		* To the File System Cache
		* To the disk
	* Upside
		* Each checkpoint is a consistent snapshot of the data, so the db won't be corrupted
		* Cause of this, could use no journaling
	* Downside
		* If no journal, could loose data which was not writent to the disk between checkpoint when the server failed
* Document-level locking
	* No locks but good concurency protocols
	* Writes scale with number of threads
* Compression : Since WT has its own cache, and since data in WT cache doesn't have to be the same as in the File System cache or on disk, it can have compression. Options :
	* Snappy (default) - fast
	* zlib - more compression
	* none

###Indexes

`db.collectionName.createIndex({a:1})`

Indexes have an order : 1 or -1 (forwards or backwards)

Following are different (two fields index = compound index) :

* `db.collectionName.createIndex({a:1,b:1})`
* `db.collectionName.createIndex({a:-1,b:-1})`
* `db.collectionName.createIndex({b:1,a:1})`
* `db.collectionName.createIndex({b:-1,a:-1})`

Can't do : `db.collectionName.createIndex({a:1,b:-1})` - indexes have **only one direction**

* Duplicate key allowed on indexes (not on _id)

####Notes

* Indexes can be any type
* `_id`
	* automatically created
	* unique to the collection
	* immutable, no change through time
* `db.collectionName.ensureIndex()` [is now deprecated in v3](http://docs.mongodb.org/manual/reference/method/db.collection.ensureIndex/)
* Indexes are automatically used when working a query
* Array content can be indexed (multikeys) : `{likes:["tennis","golf"]}`
* Nested fields or entire subdocuments can be indexed
* Fieldnames are not in the index

####Unique Indexes

`db.collectionName.createIndex( { a:1 }, { unique: true } )`

It won't be possible to insert two documents not containing the "a" attribute (will cause duplicate key error because evaluated to `null`)

####Sparse Indexes

If an attribute is used rarely but you want to index it, use sparse so that :

* it won't create an index for all the times this attribute is not present
* if an index is unique AND sparse, 2 documents not including the field indexed **can exist** in the same collection

####TTL Indexes

[doc](http://docs.mongodb.org/manual/core/index-ttl/)

`db.eventlog.createIndex( { "lastModifiedDate": 1 }, { expireAfterSeconds: 3600 } )`

* Only on **Date** (or Array of Date) attributes
* The document will be deleted after the amount of seconds specified

####Geospatial Indexes

`db.places.createIndex( { loc: "2dsphere" } )` : creates the index.

`db.places.find( { loc: { $near: { $geometry: { type: "Point", coordinates: [2,2.01], spherical: true } } } } )` : will output documents ordered by distance to this point

[More queries - doc](http://docs.mongodb.org/manual/reference/operator/query-geospatial/)

standard Based on [GeoJson](http://geojson.org/)

####Text Indexes

`db.sentences.createIndex( { words: "text" } )`

A collection can have at most **one text** index.

* non-case sensitive
* manages plurals
* words order in query doesn't matter (like OR operator)
* retrieve a score on the query and sort by pertinance
* language can be changed

Simple text query :

`db.sentences.find( { $text: { $search: "Trees cat" } } )`

Query, without showing _id and ordered by pertinance (score) :

`db.sentences.find( { $text: { $search: "Trees cat" } }, { score: { $meta: "textScore"}, _id: 0 } ).sort( { score: { $meta: "textScore" } } )`

####Background Index Creation

[doc](http://docs.mongodb.org/manual/tutorial/build-indexes-in-the-background/)

`db.collectionName.createIndex( { a:1 }, { background: true } )`:

* read and writes can continue will index is building (on the primary )
* on secondaries, indexes are not built in the background
	* any queries to the collection will block
* slower than foreground (may take time)
* foreground *packs more* (due to BTree algorithm optimizations)
* doing it in foreground : you know exactly when finished (even if it doesn't block read/writes on primary)

####Drop Duplicates

`db.collectionName.createIndex( { a:1 }, { unique: true, dropDups: true } )`

###Performance

Usefull to test indexes : `for (i=0; i<100; i++) { for(j=0; j<100; j++) { x = []; for(k=0; k<100; k++) x.push({a:i,b:j,c:k, _id: (10000*i + 100*j +k)}); db.example.insert(x); }  }`

####Explain Plans

* See which indexes are used
* Looks at the following queries :
	* aggregate
	* find
	* count
	* remove (won't remove)
	* update (won't update)
	* group

`db.example.explain().find( { a:1 } ).sort( { b: -1 } )`

In `queryPlanner.winningPlan` (don't confuse with possible `queryPlanner.rejectedPlan`), with an index:

* `"stage": "IXSCAN"`: means an index was used
* `"indexName": "a_1_b_1"`: which index was used
* `"direction": "backward"`

In `queryPlanner.winningPlan`, without index:

* `"stage": "COLLSCAN"`: means an index was used

#####queryPlanner vs executionStats

* queryPlanner : default explain
* executionStats : more infos, such as :
	* time to execute
	* number of documents returned
	* documents examined

`db.example.explain("executionStats").find( { a:1 } ).sort( { b: -1 } )`

```
var exp = db.example.explain("executionStats");
exp.find( { a:1 } ).sort( { b: -1 } )
```

#####allPlansExecution

`var exp = db.example.explain("allPlansExecution");`

Will give you the stats for all execution plans.

* The `winningPlan` is the one returning documents (other plans may not return documents because they were overuled by the winning plan, which is the one which will be used in production)
* doesn't include COLLSCAN (if index is present, it **will be used**)

**Drop unused indexes**

####Covered Queries

* Queries that can be answered without looking at documents
* Just uses the index

Use case : You retrieve exactly the same fields that are indexed (and where used in the query).

####Recap

* more indexes
	* faster reads
	* slower writes
* faster to build an index post-import than pre-import

###Ops

####currentOp() & killOp()

* `db.currentOp()`
* `db.killOp(opId)`

Which ops are safe to kill ? :

* a query **safe**
* a findAndModify **safe** (usually write operations on primary are safe)
* a foreground create index building on a primary **safe**
* a foreground create index building on a secondary **unsafe** (primary and secondary won't be in sync)
* a compact command job **unsafe** 

Summup - care killing :

* write ups to secondaries
* compact
* internal ops

###Profiler

`db.setProfilingLevel(level)`

level :

* 0 : off
* 1 : selective (slow ones)
* 2 : on

Any infos will be stored in `system.profile` collection of the db.

* No indexes on it (faster, no need because more writes than reads)
* `capped` collection that only takes 1MB by default in RAM

`db.system.profile.find().sort( { $natural: -1 } ).limit(1).pretty()`: will return the last operation profiled.

Shortcuts :

* `show profile`: will show the last op that was slow
* `db.getProfilingStatus()`
* `db.setProfilingLevel(1,3)`: so that it will only capture ops taking more than 3 ms

###mongostat & mongotop

`mongostat --port 27017`

`mongotop --port 27017`