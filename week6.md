#Week6

##Scalability

###Sharding & Data Distribution

* ReplicaSet : 
	* different members have the **same document**
	* high availability, data safety, disaster recovery ...
* Sharding : 
	* different shards have **different documents**
	* scale out

####ShardKey

* documents with same shardKey on the same shard
* ranges of shardKey
* documents with shardKey close to one another in sort order will be on the same shard (*range-based partitioning*)
	* queries wil be sent to only specific shards if the shardKey is involved

###Replication with Sharding

The more shards you have, the more likely the machines are to go down -> The more shards you have, the Replication Factor (RF) should increase.

###Chunks & Operations

`chunk`: data within the range of a shardKey (64MB by default)

* `split`: no data moved
* `migrate`: moves data from shard to shard to maintain balance between them
	* `liveliness`: data will still be readable / writable while migrating

The `balancer` decides when and where to `migrate` (built-in MongoDB).

Example : Large chunk size means chunks will be split less often as each chunks needs to grow larger before it is split.

Small chunk size vs large chunk size :

* More migrations would have to take place
* Each shard would be very balanced in terms of how much data they contain

###Sharding Processes

Multiple shards, each one containing a replicaSet, running multiple `mongod` processes.

Config servers : Small `mongod` processes storing the metadata (by default, 3 servers in prod - 1 in dev)

* one config server down : metadata can't be written any more, but still can be read
* if all config servers down : the cluster is down
* the metadata contained is important (see backups section week7)

Multiple `mongos` :

* the client connects to the cluster through one of those
* they also talk to the `mongod` processes
	* from the replicaSets in the shards (to retrieve / update data)
	* config servers (to know where to retrieve / update data)
* they have no persistent state
* doesn't need to be one `mongos` per shard, but there should be more than one `mongos` process launched

###Cluster Topology

####Demo

* Number of initial shards : 4
* Replication Factor : 3
* 3*4 = 12 `mongod` shard servers
* 4 `mongos` processes
* 3 config servers

[`run.sh` file](https://github.com/topheman/m102-notes/blob/master/run_shards.sh)

**Warning** : Change the location "imac-tophe.local" to your machine.

To run it :

```shell
$ chmod +x run_shards.sh
$ ./run_shards.sh
```

`mongos` and `mongod` processes will be up but the replicaSets won't be initiated yet.

Best practices :

* run `mongos` on the standard mongodb tcp port 27017
* do **not** run shard servers or `mongod`s nor config servers on that port

####The Config Database

Connect to the `mongos` and show the config (will retrieve it from the config server via the `mongos`).
Running `mongo` connects to the default `mongos` (on port 27017).

```
mongo
use config
show collections
```

####Adding initital Shards

For each shard :

1. Inititate replicaSet
2. *add* the shard `rs.addShard(...)`

To retrieve the port of one of the `mongod` of a replicaSet of our demo : `ps -A | grep 2700`

```
mongo --port 27000
rs.status()
rs.initiate()
rs.add('imac-tophe.local:27001')
rs.add('imac-tophe.local:27002')
rs.conf()
rs.status()
```

Connect to the `mongos` and add the replica as a shard with `sh.addShard(server:port OR replicaSetName/server:port)`

**Warn** : The server name might have a different case, since it hase been `rs.initiate()` without any config. Check the exact server name by connecting to the `mongod`and `rs.status()`.

```
mongo
sh.addShard('a/imac-tophe.local:27000')
sh.status()
```

To automate : use the [`our_setup_script.js` file](https://github.com/topheman/m102-notes/blob/master/our_setup_script.js) - replace the `imac-tophe.local`refs by yours.

```
mongo --shell our_setup_script.js
ourAddShard('b',27100)
sh.status()
ourAddShard('c',27200)
sh.status()
ourAddShard('d',27300)
sh.status()
```

At this point, the shards are all added.

####Enable Sharding for a Collection

* collections aren't sharded by default
* to specify shard keys, you need to connect to a `mongos`

1. Enable sharding on the db : `sh.enableSharding(dbName)`
	* ex: `sh.enableSharding("week6")`
	* `sh.status()` : `"partitioned" : true` means sharding is enabled
2. Enable sharding on the collection : `sh.shardCollection(fullName, key, unique)`
	* ex: `sh.shardCollection("week6.foo", { _id: 1 }, true )`

####Working on a Sharded Cluster

Prepare data - [`working_with_a_sharded_cluster.js` file](https://github.com/topheman/m102-notes/blob/master/working_with_a_sharded_cluster.js)

```
mongo working_with_a_sharded_cluster.js --shell
```

Wait for the ~1million docs to be inserted.

`db.example.getIndexes()`: an index on b,c,d fields which is the shard key (`sh.status()`)

* inserting a new document which doesn't contain the whole shard key will fail
* adding a new index will create it on all the other primaries

Query and importance of the shard key:

* targetted query: uses the shard (or at least part of it - prefix way), able to hit only a specific shard
* scatter gather query: doesn't use the shard key, will hit all shards (without necessary retrieving any data from them)

###Cardinality & Monotonic Shard Keys

####Shard Key Selection

* the shard key should be common in queries for the collections
* good "cardinality" / "granularity"
* is the key monotonically increasing ?
	* like timestamp / autoincrement / ObjectId in _id field - yes ? don't pick this field
	* use UUID

####Examples

Orders collection with the following fields :

* _id
* company
* items : []
* date
* total

Possible shardKeys :

* _id : good cardinality, used in query, bad monotonic
* company : bad cardinality
* compound shardKeys :
	* company, _id : if query only on _id, useless
	* company, date : great for query, great for spinning accross shards

###Process and Machine Layout

* shard servers : `mongod --shardsvr`
* config servers : `mongod --configsvr`
* `mongos` processes

Use cases with 32 machines / RF (Replication Factor) = 3

* 10 shards * 3 (RF) = 30 machines with a `mongod` running, part of a replicaSet
	* 3 config servers on any of those machines (not large load)
		* don't use shard 1 (might be used more than others)
	* `mongos` : different options :
		* one on each client machine
		* one on each of the two spare machine (2 `mongos` won't be enough for a cluster that large)
		* one on each member of the cluster :
			* clients will connect to any `mongos`
* 8 shards * 2 (RF) = 16 machines with a `mongod` running, part of a replicaSet
	* *never have even number of member* : add arbiter
		* add an arbiter on shards, for each replicaSet, not located on the same shard
	* `mongos` running on each machine
	* 3 config servers each ones, located on different machines
* Very large cluster
	* dedicated machines for `mongod`, config, `mongos`
	* spare servers (when machines go down)

###Bulk Inserts & Pre-splitting

Generally, Pre-splitting shouldn't be necessary, even with non uniform key distributions - the database handles the split / migrate.

###Further Tips & Best Practices

* only shard the big collections (don't bother with smaller ones)
* pick shard key with care, they aren't easily changeable (create new collection and copy data over)
* pre-split if bulk loading
* be cognizant of monotonically increasing shard keys (timestamp / ObjectId ...)
* adding a shard is easy but takes time
* use logical server names for config servers
* don't directly talk to anything except mongos except for dba work
* keep non-mongos processes off of 27017 to avoid mistakes
