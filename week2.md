#Week2

##CRUD

###Inserting

* `db.sample.insert({a:1})`
* `db.sample.insert({a:1})`
* `db.sample.insert({a:2})`
* `db.sample.insert({b:1})`
* `db.sample.insert({a:7})`
* `db.sample.insert({a:2,b:"Hello world"})`

`db.sample.find().pretty()`

Write error management : see later :

* db.getLastErrror()

###Update

* Signature : `db.collectionName.update(whereClause, documentOrPartialUpdate [, upsert [, multi]])`
* Two kinds of updates :
    - Full document update
        + _id are immutable (must exist / unique / index)
    - Partial update (ex: change single field)
* `db.collectionName.save(objectRetrievedAndModified`: helper to update (so that won't have to pass again the where clause with `_id`) **shell helper**
    * If `objectRetrievedAndModified` has no id, it will be inserted (an `_id` will be automatically created for it)

###Partial update & Document limits

* Limit per document : 16 MB
* [Special operators](http://docs.mongodb.org/manual/reference/operator/update/)
    - `$set`
    - `$push`
    - `$addToSet`
    - `$pop`
    - `$unset`
    - ...

###Removing document

`db.collectionName.remove(whereClause)`: Multiple remove (according to the where clause)

###Multi Update

new Syntax (since mongo 3.x.x):

`db.collectionName.update(whereClause, updateDocument, [options])`

options:

```
upsert: true/false,
multi: true/false,
writeConcern: document
```

By default :

* upsert=false -> won't create any new document
* multi=false -> will only update the first matching document
	* interesting when you know there is only one matching line (not to continue scanning the entire db)
	* avoid updating the whole db
	
###Upsert

Insert if not present when updating

* Avoids checking if line exists before updating (then choose between insert and update)


##Administrative Commands

###Bulk write operations

Grouped requests ordered or unordered - [docs](http://docs.mongodb.org/manual/reference/method/Bulk/)

```
var bulk = db.collectionName.initializeUnorderedBulkOp();
bulk.insert({...});
bulk.insert({...});
bulk.insert({...});
bulk.execute();
```

###Commands

[Common commands](http://docs.mongodb.org/manual/reference/command/) :

* user
	* getLastError
	* isMaster
	* aggregate
	* mapReduce
	* count
	* findAndModify
* DBA
	* drop
	* create
	* compact
	* serverStatus
	* replSetGetStatus
	* addShard

Some of those commands are run on the `admin` db (reserved name) - called "Admin commands"

**How to run a command ?** : `db.runCommand({commandName:value, param1:value, ...})`

###db.isMaster()

Can be run :

* `db.runCommand({isMaster:1})`
* `db.runCommand("isMaster")`
* `db.isMaster()`

*Determines if this member of the replica set is the Primary*

###db.currentOp() / db.killOp()

*Returns a document that contains information on in-progress operations for the database instance.*

Usefull to detect performance problems.

Each operations has an id and can be killed with `db.killOp(opId)`.

###collection.stats() & collection.drop()

`db.collectionName.stats()` returns infos about the collection such as :

* number of lines
* size
* index size
* ...

`db.collectionName.drop()` will delete the collection itself.

##Usual Commands by level

* server
	* isMaster
	* serverStatus
	* logout
	* *getLastError* (now implicitly called on v3 shell)
* db
	* dropDatabase
	* repairDatabase
	* clone
	* copydb
	* stats
* collection
	* create
	* drop
	* stats
	* renameCollection
	* *count*
	* *aggregate*
	* *mapReduce*
	* *findAndModify*
	* *geo* *
* index
	* ensureIndex
	* dropIndex

## Usefull Links
[Kill all long running MongoDB queries at once](http://cacodaemon.de/index.php?id=65)

```js
(function (sec) {db.currentOp()['inprog'].forEach(function (query) { 
    if (query.op !== 'query') { return; } 
    if (query.secs_running < sec) { return; }  

    print(['Killing query:', query.opid, 
           'which was running:', query.secs_running, 'sec.'].join(' '));
    db.killOp(query.opid);
})})(120 /*The maximum execution time!*/);
```
