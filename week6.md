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

To run it :

```shell
$ chmod +x run_shards.sh
$ ./run_shards.sh
```

`mongos` and `mongod` processes will be up but the replicaSets won't be initiated yet.

Best practices :

* run `mongos` on the standard mongodb tcp port 27017
* do **not** run shard servers or `mongod`s nor config servers on that port

###The Config Database

