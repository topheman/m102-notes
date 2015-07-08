#Week7

##Backup & Recovery

###Security

####Overview

Depends on where you run MongoDB :

* Trusted environment
* MongoDB Authentication (good practice : use SSL atop : compile with `--ssl` - [link](http://docs.mongodb.org/manual/tutorial/configure-ssl/))
	- `--auth` : securing client access
	- `--keyFile` : intra-cluster security

####Authentication & Authorization Overview

[Docs](http://docs.mongodb.org/manual/security/)

* Authentication
	* MongoDB CHALLENGE/RESPONSE (original protocol)
	* X.509 / SSL
	* KERBEROS or LDAP (Enterprise Edition)
* Access control / authorization
* Encryption
* Network setup
* Auditing

####Authentication

Turn on security (authentication) : `mongod --auth` or `mongos --auth`

When you connect with mongo shell from localhost **the first time**, no authentication necessary - once setup, it will need a password.

```
use admin
var me = {
  "user": "tophe",
  "pwd": "azerty",
  "roles": [
    "userAdminAnyDatabase"
  ]
}
db.createUser( me )
```

`db.createUser( user, writeConcern )` : [doc](http://docs.mongodb.org/manual/reference/method/db.createUser/)

Then disconnect (since no operation will be possible as the current shell isn't logged) and reconnect : `mongo localhost/admin -u tophe -p`

This user won't be able to read the collection data : will have to create an other one withe the role : `readWriteAnyDatabase` - [see roles](http://docs.mongodb.org/manual/reference/built-in-roles/).

Standard roles available:

* Specific Database
	* `read`
	* `readWrite`
	* `dbAdmin`
	* `userAdmin`
	* `clusterAdmin`
* All Databases
	* `readAnyDatabase`
	* `readWriteAnyDatabase`
	* `dbAdminAnyDatabase`
	* `userAdminAnyDatabase`
	
What are the means of `--keyFile` ?

To use in a cluster context so that mongods know how to authenticate themselves, the keyFile would point to a secret

####SSL and KeyFile

`--keyFile` give us authentication for members of the cluster. It doesn't encrypt your data, it just makes sure that the servers authenticate one another using the keyfile.

By default, data between client and members and between members themselves is not encrypted. You can use SSL to encrypt by compiling MongoDB with the `--ssl` flag from sources (`scons --ssl`).

####Security and Clients

Type of Users (clients) :

* `admin`:
	* can do administration
	* created in the `admin` database
	* can access all databases
* regular users:
	* access a specific db
	* read/write or readOnly

####Intra-cluster Security

Keyfile has to be base64 (confirm ?).

```
openssl rand -base64 60 >> secret.txt
mongod --dbpath data --auth --replSet z --keyFile secret.txt
```

###Backups

####Overview

Methods for an individual set/server:

* `mongodump`
* filesystem snapshot
* backup from secondary
	* shut down, copy files, restart

####Mongodump

* create the backup: `mongodump --oplog`
* restore the backup: `mongorestore --oplogReplay`

#####Filesystem Snapshoting

* need journaling enable - keep consistancy

####Backup a Sharded Cluster

Backup each shard individually

1. Turn off the balancer `sh.stopBalancer()`
2. Backup the config db : `mongodump --db config`
3. Backup each shard replica set (individually)
4. Turn balancer back on `sh.startBalancer()`

####Backup Strategies

```shell
# back up a sharded cluster

mongo --host some_mongos --eval "sh.stopBalancer()"
# Make sure that worked !

# back up config database / a config server
mongodump --host some_mongos_or_some_config_server --db config

# back up all the shards
mongodump --host shard1_svr1 --oplog /backups/cluster1/shard1
mongodump --host shard2_svr1 --oplog /backups/cluster1/shard2
mongodump --host shard3_svr1 --oplog /backups/cluster1/shard3
mongodump --host shard4_svr1 --oplog /backups/cluster1/shard4
mongodump --host shard5_svr1 --oplog /backups/cluster1/shard5
mongodump --host shard6_svr1 --oplog /backups/cluster1/shard6

# balancer back on
mongo --host some_mongos --eval "sh.startBalancer()"
```
Keep in mind :

* Check cluster is healthy before begin (data grabbed from specific servers in replica sets)
* Do the operation in parallel


###Additional Features of MongoDB

####Capped Collections

* prealocated max size
* circular queues / least recently inserted order (order is the same as insertion order)
* cannot delete / update document

####TTL Collections

* auto age-out of old documents
* creates special index

####GridFS

Grid FileSystem

BSON size limit : 16 MB

Use case: store large blob

Utility : `mongofiles`

####Hardware Tips

[Production Notes](http://docs.mongodb.org/manual/administration/production-notes/)

* Fast CPUs clock > more CPU cores
* RAM is good
* 64 bit
* Virtualization is ok (not required)
	* some problems with `openvz`
* Disable NUMA ( [Non Uniform Memory Access](https://en.wikipedia.org/wiki/Non-uniform_memory_access) )
* SSDs are good (moreover with read + write)
	* reserve empty space (unpartitioned) ~20%
* filesystem cache is most of mongod's memory usage
* check [readahead](https://en.wikipedia.org/wiki/Readahead) settings (small value)

###Additional Resources

* [docs.mongodb.org](http://docs.mongodb.org/manual/)
* [driver docs](http://docs.mongodb.org/ecosystem/drivers/)
* [bug database / Feature](https://jira.mongodb.org/)
* support forums
	* [mongodb-user / google groups](https://groups.google.com/forum/#!forum/mongodb-user)
* [blog.mongodb.org](http://blog.mongodb.org/)
* [@mongodb](https://twitter.com/mongodb)
* [MMS](https://www.mongodb.com/cloud)