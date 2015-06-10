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
	* Journal : 