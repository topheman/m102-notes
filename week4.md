#Week4

##Replication

###Overview

* Multiple redundent copies of the same document
* Reasons :
	* "HA" : High Availability (fail over)
	* Data Safety (durability)
		* Extra copies / Backup
		* "DR" : Disaster Recovery
	* Read preferance (scalability ... - more later)

###Asynchronous Replication

* single primary
* multiple secondaries (slaves)

* works on commodity hardware (example RAID)
* works accross wirde area network
* provide eventual consistency *(last writer wins)*

###Statement Based vs Binary Replication

A little bit of both : **statement based replication - simplified**

On primary : `db.foo.remove( { age: 30 } )`

On secondary :

```
db.foo.remove( { _id: 123 } )
db.foo.remove( { _id: 564 } )
db.foo.remove( { _id: 486 } )
db.foo.remove( { _id: 654 } )
db.foo.remove( { _id: 555 } )
db.foo.remove( { _id: 777 } )
db.foo.remove( { _id: 844 } )
db.foo.remove( { _id: 899 } )
...
```

###Replica Sets Concepts

A *Set* is also called a *Replication cluster*. It includes :

* A primary
* One or more secondaries (there can be no secondaries, but not usefull)

Shards : Each shard is a replica set.

1 primary, 2 secondary =

* 3 members
* RF (Replication Factor) = 3

####Automatic Failover

* All members of a set monitor each others
* When the primary goes down :
	* each member notice it
	* they have a **consensus** about it (**majority** of the members are ok to sey that the primary is down in that case)
	* they elect a new primary
	* the driver (which is replicaset-aware) will send its writes to the new primary
		* writes are **always sent to primary**
		* reads can also be retrieved from secondaries (see more in ReadPreference)

####Automatic Node Recovery

When a primary goes down, since replication is asynchronous, some writes can be commited on the primary but never make it through the secondaries. So when the secondary is elected primary, it hasn't those writes and new data keeps comming.

Before the primary that went down comes up back as a secondary: There is an automatic rollback of the uncommited writes that didn't make it through to the secondary before it wen't down as a primary, to get even with the state the new primary started:

* Automatic rollback uncommited writes on primary which went down before getting up to date with the new elected primary (and archive them)

*Notes: Not easy do describe only in words ...*

###Replica Sets

####Start a Replica Set

* Replica set have name so that members know which one to join / not to join

```
cd db
mkdir 1 2 3
mongod --port 27001 --replSet abc --dbpath /Users/.../db/1 --logpath /Users/.../db/log.1 --logappend --oplogSize 50 --smallfiles --fork
mongod --port 27002 --replSet abc --dbpath /Users/.../db/2 --logpath /Users/.../db/log.2 --logappend --oplogSize 50 --smallfiles --fork
mongod --port 27003 --replSet abc --dbpath /Users/.../db/3 --logpath /Users/.../db/log.3 --logappend --oplogSize 50 --smallfiles --fork
```

* `--replSet`: attaches mongod to this replicaSet
* `--logappend`: so that when restarted, the log doesn't get overwritten
* `--oplogSize`: size of the oplog in MB (default: 5% of available free space on disk, not more than 50GB)
* `--smallfiles`: keep the datafiles smaller (see doc ?)

The replicaset will be ready to be initialized.

*Shards*

If you have a shard named "abc", you would name each replicaSet on the shard "abc1", "abc2", "abc3"

####Initiate a Replica Set

Use case: maybe launching a replicaset on a previously mongod which has already data (or not)

* no previous data: connect to any member
* previous data: connect to the member containing the data `--replSetInitiate` or in the shell : `rs.initiate( config )`

Use case: no previous data

```
mongo --port 27001
cfg = {
  _id: "abc",
  members: [
  	{ _id:0, host: "imac-tophe.local:27001" },
  	{ _id:1, host: "imac-tophe.local:27002" },
  	{ _id:2, host: "imac-tophe.local:27003" }
  ]
}
rs.initiate(cfg)
```

Best practices:

* don't use raw ip addresses
* don't use names from `/etc/hosts`
* use dns
	* pick an appropriate TTL ( ~5 minutes)
	
####Replica Set Commands

* `rs.conf()`: returns the config of replica set
	* config can be found in collection `db.system.replset`
		* only one single document
		* read only, to change it, use : `rs.reconfig(config)`
* `rs.add(host)`: ad new member to set
* `db.isMaster()`: retrieve infos
* `rs.stepDown(timeInSecs)`:  stop being primary for a number of seconds
* `rs.freeze(timeInSecs)`: prevent a node from becoming primary for a number of seconds

####Reading and Writing

Reads on secondary disbled by default - to enable : `rs.slaveOk()`

####Failover

* Shut down cleanly : `kill pid`
* Shut down like it crashed : `kill -9 pid` **do not use -9**

####Read Preference

*aka slaveOk*

* In shell: `rs.slaveOk()`
* In driver:  "Read Preference"

#####Why query secondary ?

* geography
* seperate a workload (analytics / reporting)
* seperate load (better do scaling)
* availability (if primary down) - keep in mind you may get **stale data** (eventually consistency)

#####Options

* `primary` (default):
	* consistent reads
	* no stale data
	* *primary only*
	* *to use if doubt*
* `primary Prefered`: fallback to secondary if can't reach primary
	* to keep the load off the primary
	* *primary possible*
	* *secondary possible*
* `secondary`
	* *secondary possible*
	* *to use for certain reporting workloads*
* `secondary Prefered`: use secondary first, fallback to primary if can't reach any secondary
	* *primary possible*
	* *secondary possible*
* `nearest`
	* *primary possible*
	* *secondary possible*
	* *to use if remote data center*
	* *consider for even read loads*
