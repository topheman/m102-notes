#Week1

##Concepts

MongoDB started due to :

* **scale** : hardware evolution 
	* **parallelism** (cores/servers)
  	* cloud 
  	* need for scalability (big data)
* **development** method evolved (relational database are 40 years old)
* **complex** data structure
  	* unstructure
  	* polymorphic

##Scaling

###Vertical scaling vs Horizontal scaling

* Horizontal scaling :
  	* more servers
  	* more network dialog
  	* => one server is more likely to go done
    	* need to deal with failure / latency (hard to do)
* Vertical scaling :
  	* Working or not working (one part fails, the whole part fails)

###Scale Out

* The less features you have, the more you can scale
* The more features you have, the less you can scale

Memcache -> [RDBMS (Relational Database Management System)](http://en.wikipedia.org/wiki/Relational_database_management_system)

Mongo tries to find a good balance between scale+speed/features.

##SQL and Complex Transactions

Joins / Complex Transactions

* **joins** : Data comming back from multiples tables spread on multiple machines after a query
  	* time to send a message between servers is significant -> **dealbreaker**
* **transactions** : Query touching data on multiple rows in multiple servers, needing atomicity (potentical rollback if transaction fails)
  	* doesn't scale up with speed -> **dealbreaker** 

**Choose not to do both of those things in MongoDB**.

##Document overview

**No joints -> no relational -> different data model.**

**Document Oriented** :

* Format : JSON (internally BSON)
* Basic unit of storage

Mongo query language also based on JSON.

##Installing

```shell
$ brew update
$ brew install mongodb
$ sudo mkdir -p /data/db
$ sudo chmod 777 /data
$ sudo chmod 777 /data/db
```

##JSON Types

[JSON](http://en.wikipedia.org/wiki/JSON) : JavaScript Object Notation

**BSON** actually used in MongoDB

Document Oriented Data Base

JSON types : Number / Boolean / String / Array / Object / null

##BSON

**Binary JSON** : http://bsonspec.org

* fast scannability
	* skip to key of interest without scanning all others
* data types
	* Date datatype
	* Binary datatype (no need to use base64)
	* ObjectId (used by default for the _id)
* More compact than JSON (most of the time).
* Data Type can be very efficient on operations such as increment for example.

##BSON in application

* Documents are stored on disk in BSON
* They go over the wire in BSON
* The driver (whatever the language) retrieves BSON
* The driver translates BSON to language local usable objects (such as JavaScript Objects or Java Objects ...)

There are utilities to directly manage BSON (such as mongoimport).

##Dynamic Schema

We DO scheme design, accurate called 'dynamic schema'. MongoDb is more like a dynamically typed system, types resolved at runtime.

Flexibility

* iteration / agile
* data representation / polymorphic

## MongoDB shell

* Connects always to database 'test' on default

###Launch the database :

```shell
$ mkdir data
$ mongod  [--nojournal] --dbpath data
```

##Mongoimport

###Launch the database :

```shell
$ mkdir data
$ mongod --dbpath data
```

Take a look at the data to import : `cat homework/week1/homework_1_2/products.json` : **No trailing commas ?!!** WTF, it isn't valid JSON ...

It is said :

* JSON docs should be one lined
* No commas at the end (at least it's allowed, didn't check if it works with commas)

###Import

`mongoimport [--stopOnError] --db pcat --collection products < homework/week1/homework_1_2/products.json`

* Databases and collections are automatically created
* Database files (in the dbpath) has max. 2GB size, automatically created new one
* "_id" will be added automatically to each document, if not specified

###Cursors introduction

* `db.products.find()` : in the repl, retrieves results ten by ten
* to get the next one : `it`

`db.collectionName.find(query, projection)`:

* projection lets you show or not some fields : `db.products.find({},{name:true,_id:false})` (only shows names)

###Query language

####Basic Concepts

* mongo queries represented as BSON

####Projection

Second param of `find` or `findOne` are field selectors (like methods)

* `{name:1}`: will show _id and name fields
* `{name:1, _id:0}`: will show only name fields
* `{limits:0}`: will show any fields but limits
* `{limits:0, _id:0}`: will show any fields but limits and _id

####Schemaless

* fields can vary
* datatypes can vary
* no 'alter table'
* reduces the cases where database scheme migrations are needed

Why use different fields ?

* polymorphism
* agile / iterative

####Sorting

`db.collectionName.find(...).sort(params)`

Examples of params : 

* `{price:1}` : price ascending
* `{price:-1}` : price descending
  	* document without `price` attribute evaluate to `price = null` (null less than any number)
  	* might use `db.collectionName.find({price:{$exists: true}}).sort({price:1})`

####Cursors

* Results are returned in batch of reasonable size
* Results are consumed transparently via the driver
  	* Not expansive to `db.test.find()` on ten millions docs (will only return like the 20 first) and only give the following on demand
  	* `db.test.find({x:190000})`: more expansive beause lots of docs scanned before getting to this one
  	* `db.test.find().skip(20).limit(5)`:
    	* skip: not free because docs are scanned even if not sent
* `db.test.find().sort({x:-1}).skip(20).limit(5)`: applied before query sent to server (**not** managed client side)

###Next

Different ways to use MongoDB

* Standalone
  	* Simpliest way to use MongoDB
  	* One `mongod` answering to your app via the mongo driver (will use mongo shell)
  	* Advantages
    	* simplicity
    	* cost
  	* Disadvantages
    	* No redundancy
    	* Doesn't scale
* Replica Set
  	* Same as previous: the app dialogs via the mongo driver
  	* **Not only one** mongod, but multiple mongod
    	* primary
    	* secundaries
  	* Disadvantages
    	* More complex
    	* Still doesn't scale
  	* Advantages
    	* Data durability : Server goes done ? Got more copies
    	* Availability : Server goes done ? An other takes its place
* Sharded Cluster
  	* Data split into multiple shards (which are replica set)
  	* Disadvantages
    	* Cost (multiple servers)
    	* Complexity
  	* Advantages
    	* Scaling
    	* Durability
    	* Availability
  	* "MongoDB" is from *humongous Data Base* http://en.wikipedia.org/wiki/MongoDB
