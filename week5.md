#Week5

##Replication - Part2

limits per sets :

* no more than 12 members
* no more than 7 voters

###Reconfiguring a Replica Set

`rs.config()` : The `version` field lets members of the set know if they are up to date with the config.

Use case :

* A member goes down
* The config changes
* The member goes back up
* It notices it hasn't the latest version of the configuration and updates it
* Then makes recovery

A reconfiguration will:

* trigger an election
* members can be removed or added
* no need for all member a replica set to be available to reconfigure

###Arbiters

`rs.config()`

```
{
	"_id": 0,
	"host": "toto.local:27001",
	"arbiterOnly": false
}
```

Arbiter :

* Server with **no data**
* Does participate in election

Use case: "Split brain problem" :

* 2 servers with data
* One of them is cut down from the world (its network connection dies)
* An election goes on to vote for a new primary
* Both servers will vote for themselves, we'll need a tie-breaker
* With an arbiter, it will vote for the reachable one (making a vote majority)

* Protects agains network split
* Lets you making odd number of votes in replica set
* Lets you spread the replica set over more data centers


###Priority

`rs.config()`

```
{
	"_id": 0,
	"host": "toto.local:27001",
	"priority": 1
}
```

* 0: never primary
* Any other positive number (1, 2.8, 500 ...): elligible for primary

###Hidden Options & Slave delay

`rs.config()`

```
{
	"_id": 0,
	"host": "toto.local:27001",
	"hidden": false,
	"slaveDelay": 0
}
```

* `hidden:true`:
	* clients won't be able to directly query it
* `slaveDelay:8*3600`:
	* the member will be 8 hours behind the primary (should also be `hidden:true` since the data won't be fresh)
	* can be used to rollback operations without using backup

Usefull for :

* prevent against new client application release bug
* getting a view of the DB between backups
* during dev when using experimental queries

Note: You can now do a snapshot of your data periodically and use oplog for a point-in-time restore, so **slaveDelay is perhaps not as useful as it once was**.


###Voting options

`rs.config()`

```
{
	"_id": 0,
	"host": "toto.local:27001",
	"votes": 1
}
```

**Don't use votes**

###Write Concerns

####Principles

* Write is truly commited upon application at a majority of the set
* We can get aknowledgment of this

* `db.students.update( { _id : 3 }, { $set : { grade : "A" }, { writeConcern: { w : "majority" } } )`: majority of the nodes
* `db.students.update( { _id : 3 }, { $set : { grade : "A" }, { writeConcern: { w : "all" } )`: all nodes
* `db.students.update( { _id : 3 }, { $set : { grade : "A" }, { writeConcern: { w : 1 } } )`: only primary
* `db.students.update( { _id : 3 }, { $set : { grade : "A" }, { writeConcern: { w : 0 } } )`: fire and forget

####Use cases

* `w:0`: logging, page view counters, analytics ...
	* call every N
* `w:majority`: most cases - garanties data
* `w:all`: flow control (to ensure not starting next step until every member are up to date, like an import)
* `w:1`: write availability (ex : dupKey notification) - not on critical data

**Call every N pattern** (also known as `w:<tag>`):

```
var N = 1000000;//number of docs to insert
for(var i=0; i<N; i++) {
  db.foo.insert( arr[i] );//insert from arr
  //each 500th times AND the last time - we ensure no problem happened (if using "all", we could even ensure secondary member catch up)
  if( i%500 === 0 || i === N-1) {
  	getLastError("majority");
  }
  else {
  	getLastError(1);//all other time, only ensure write to primary
  }
}
```

Downside : In sharded environment, an error reported by `getLastError` won't mean that all the writes were down (since they probably were not sent to the same replica set because of the shard keys).

**Batch inserts**: call `getLastError()` at the end of the job

**Secondaries lagging**: *In previous versions of MongoDB where drivers didn't support non-blocking I/O*

*Connection pile-up in slowness*: If there is a peek of connections, doing a lot of writes, the secondaries might be lagging and won't aknowledge inserts/update operations immediatly but a few seconds later (which will be too late).

* choose wisely max poolsize (max connection per repl)
* choose wisely `wtimeout`
	* each query might take longer so, connections might pile-up
* monitor for lag (don't let the clients wait too long for aknowledgment with a too big `wtimeout`)

###Replica Sets in a Single Datacenter

* 3 members
* 2 + 1 arbiter
* 2 with manual failover
* 5 members (not very effective - 4 secondaries is heavy)
* 2 large + 1 small (one of the secondaries has `priority:0` - will never be primary, maybe less powerfull, can't keep up and lags)
	* in that case, wouldn't use `w:majority` (if the powerfull secondary goes down, the repl will lag)

###Replica Sets in Multiple Datacenters

* DC1 : 2 members / DC2 : 1 member `priority:0` (disaster recovery server)
* DC1 : 1 member / DC2 : 1 member / DC3 : 1 member
* DC1 : 1 member / DC2 : 1 member / REMOTE : 1 arbiter
* Practices :
	* add arbiter to have an odd number of voters
	* set `priority:2` on the most suited server to be primary
	* set `priority:0` on the less powerfull servers to avoid them being primary

###Replica Sets and Storage Engine Considerations

Replication sends **operations** from primary, **not bytes**.

Use cases:

* Testing
* Upgrading the storage engine - http://docs.mongodb.org/manual/release-notes/3.0-upgrade/
	* entire data set will go over the wire (then the primary will have to read it entirely)
