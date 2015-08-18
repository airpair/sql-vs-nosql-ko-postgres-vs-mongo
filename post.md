Synopsis 

>This article will compare and contrast one of the most commonly used relational databases (Postgres) with one of the most commonly used document databases (Mongo). It will look at the strengths and weaknesses of both, and try to make practical recommendations for when to use which. It is by no means a comprehensive survey of the database landscape. Entire families are not represented (graph databases for one). 
>
>This article is going to assume a relatively high level of general technical ability. Some understanding of JSON would be advantageous. Knowledge of databases is not required, as background theory will be explained. If you already have a solid understanding of database theory, the first chapter can be skipped with no loss of information. The second chapter will compare, at a high level, the features provided by both databases, and attempt to resolve this battle once and for all! In the interests of continuing the whimsy, when a round ends with a clearly dominant winner, I will declare it by knockout, but when it is slightly less clear cut, I will call it a technical decision, and include some justification. 

## 1. Theoretical Background 

In this chapter we will talk about the theoretical background necessary for a serious discussion of databases. Everything discussed here, will be assumed as background knowledge throughout the rest of this article. 

### 1.1 The CAP theorem. 

Any serious discussion of databases will include at least passing reference to the CAP theorem. The CAP theorem states that a networked shared-data system can have at most two of the following three guarantees: consistency (at any given point, every node sees the same data, and the data is up to date) availability (every request is correctly notified by every node), partition tolerance (The system functions irrespective of network partitioning). This theorem is further used to loosely classify databases into three families, CA, CP, and AP. To represent database systems that fulfill two of these criteria. CA systems are patently impossible, as the introduction of even a single networked client renders the system partitioned. Any vendor selling a CA system is lying outright. CP systems will, in the presence of a partition, choose to serve only consistent data. If data consistency can not be ensured, they will refuse to service the request. AP systems will, in the presence of a partition, continue responding, but will make no guarantees of the consistency of the response. 

### 1.2 Transactions 

A database transaction is a representation of a single unit of work against the database, to be processed as such. 

#### 1.2.1 ACID Transactions 

ACID stands for: atomic(Everything in the transaction is part of a single 'atom'. Either everything succeeds or everything fails.) consistent(Both the starting and finishing state of the transaction are valid and do not violate any database rules) isolated(Transactions are isolated from each other and can not interfere with each other) durable(All completed transactions persist permanently). The consistency in CAP is in fact a much stronger guarantee, and is commonly referred to as linearizability. 

#### 1.3 Linearizability 

The real meaning of the C in CAP. Informally, the guarantee is that once an operation completes, all later operations, irrespectively of transaction or node, should see a persistent, durable result of that operation, or a later state. Simply put, if we've written to variable `x`, any and all alter operations must pull the correct value from `x`.

#### 1.4 Normalization 

Wikipedia defines normalization as "the process of organizing the attributes and tables of a relational database to minimize data redundancy." There are many normalization strategies, but a database is usually considered normalized if it is in third normal form (3NF). What this means is the following: 

1. Each attribute contains only atomic values. This explicitly rules out complex JSON or array structures stored at the leaf node. 

2. No data is redundantly represented based on any non unique subsets. Put differently, for every unique set of entries (this is called a candidate key), no other attribute depends on any subset of the candidate key. 

3. No data is dependent on anything other then the key. Some examples: Let's take the following simplistic table: 

<table>
  <thead>
    <tr>
      <th> dob</th> <th> name </th> <th> attributes </th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td> 1/1/1936 </td><td> Edsger </td> <td>{hair: blue, eyes: green} </td>
    </tr>
    <tr>
      <td> 2/2/1937 </td><td> Alan </td><td> {hair: green, eyes: purple} </td>
    </tr>
  </tbody>
</table>

This table fails the first condition (commonly referred to as first normal form or 1NF), as the attributes column is non atomic, e.g. represents multiple pieces of information per value. Now lets take a table that passes that criteria but fails the second (commonly referred to as second normal form or 2NF). 

<table>
<thead>
<tr>
<th> specialty </th><th>name</th><th>eye_color</th><th>hair_color</th>
</tr>
</thead>
<tbody>
<tr>
<td> databases </td> <td> Edsger </td> <td> blue </td> <td> green </td>
<td> programming </td> <td> Alan </td> <td> green </td> <td> purple </td>
<td> algorithms </td> <td> Alan </td> <td> green </td> <td> purple </td>
<td> algorithms </td> <td> Edsger </td> <td> blue </td> <td> green </td>
</tr>
</tbody>
</table>

This fails the condition, since neither the specialty nor the name are sufficient to uniquely identify a row, however the eye and hair color depend *only* on the name, thus leading to duplication of information. Finally lets take a look at the following table: 

<table>
<thead>
<tr>
<th> specialty </th> <th> name </th> <th> company_worked_at </th> <th> company_address </th>
</tr>
</thead>
<tbody>
<tr> <td> databases </td> <td> Edsger </td> <td> Foobar LTD </td> <td> 1 Street </td></tr>
<tr> <td> programming </td> <td> Alan </td> <td> Foobar LTD </td> <td> 2 Street </td></tr>
<tr> <td> algorithms> </td> <td> Alan </td> <td> Fubar GMBH </td> <td> 3 Street </td></tr>
<tr> <td> algorithms </td> <td> Edsger </td> <td> Blue Corp </td> <td> 4 Street </td> </tr>
</tbody>
</table>

This table shows us which companies the experts have worked at and in which capacity, however the addresses refer to the company rather then the expert or their specialty, this violates 3NF. Put simply, this can be remembered as "Everything must provide a fact about the key, the whole key, and nothing but the key, so help me Codd". (Codd being the scientist who coined these forms.) Now at this point a very reasonable question to ask might be "Why is this useful?" and "How does this apply in practice". In short, by normalizing data, we ensure that data is not duplicated, we ensure that the Postgres query optimizer can make the best possible decisions (the exact algorithm for this is outside the scope of this article), and we also ensure that we don't have corrupted data anywhere in our system (due to abandoned join tables for example). 

#### 1.5 Relational Databases (RDBMS) 

A Relational Database Management System (RDBMS) is a database based on the relational model. It is designed to focus on relations between data, and stores data in tables of rows of columns. A detailed discussion of the mathematics behind the relational model is out of scope for this article, but the concepts of normalization apply to RDBMS to a very large degree. 

#### 1.6 Document Databases 

Are designed to store what is known as semi-structured data, data where there is no clear separation between the data's schema (think of this as the shape of data, what relations it has, how many columns, etc), and the data itself. 

#### 1.7 Indexing 

Wikipedia defines a database index as: 
>A database index is a data structure that improves the speed of data retrieval operations on a database table at the cost of additional writes and storage space to maintain the index data structure.`

https://en.wikipedia.org/wiki/Database_index 

More loosely speaking, a database index is a way for a database to keep track of the values of a particular column or attribute, thus making said column or attribute much faster to read or search for, but slower to write to. 

## 2. The Fight 

### 2.1. Classification (the contestants). 

#### 2.1.1 Classification of Postgres (in the right corner). 

Postgres is an RDBMS with ACID compliant transactions, up-to and including full serializability of transactions if the transaction level is set to be `Serializable`. Its Query language is a flavor of `SQL`, and it is CP, as is demonstrated here: 
https://aphyr.com/posts/282-call-me-maybe-postgres. 

#### 2.1.2 Classification of MongoDB (in the left corner) 

MongoDB is a multi storage engine beast. The default storage engine is MMAP, though WiredTiger is a newly released storage engine that purports to address some of the more systemic flaws. However as evidenced by the large number of bugs related to both data loss and memory leakage, it is clearly not yet ready for prime time. As such, while improvements purported by WiredTiger will be called out and analyzed where possible, the contender in the left corner is MMAP. MongoDB is a schema-less, document oriented DBMS with JSON like objects forming the model. It does not support transactions out of the box. It is not AP because it is a single-master system, and all reads to to the primary node (although this can be ameliorated with Replica Sets and automatic failover, whereby a new primary is selected on network partition). If the primary node is down, It is also not CP as its consistency claims are disproven here:

https://aphyr.com/posts/322-call-me-maybe-mongodb-stale-reads 

### 2.2 Consistency and Transactions (Round 1): 

#### 2.2.1 Transactions in Postgres 

Postgres does not require read locks since every transaction has a snapshot of the database. An inconsistent read (also known as a dirty read) is therefore impossible. Postgres has 3 levels of transaction isolation. For an in depth discussion, look at the documentation here:

https://www.postgresql.org/docs/9.1/static/transaction-iso.html 

The first is `Read committed`. This is the default. What this means is that each query within the transaction only sees data that had already been committed to the database when that query began. No changes made during the execution of the query will be visible, however concurrent changes that are written during between queries by other transactions *will* leak into the transaction. 

The second is `Repeatable Read`. This transaction level guarantees that no changes made by concurrent transactions during the transaction execution will be visible. In effect, the transactions changes are applied onto a snapshot independently of other transactions. If a transaction fails due to a different transaction having modified data the transaction will fail with a concurrency exception, and the application you are writing will have to retry the transaction. 

The final is `Serializable`. This transaction level guarantees that any transactions executed concurrently will have the exact same effect as if they had been executed one after the other. While this may appear on the surface to be very similar to `Repeatable Read` there is a great explanation of the difference here: https://wiki.postgresql.org/wiki/SSI 

A short (practical) example (taken from the above documentation) is a bank which allows you to overdraw any one of your account as long as the cumulative sum in all of your accounts is above the negative. A malicious attacker might attempt to exploit `Repeatable Read` by withdrawing large sums from all accounts concurrently. Since `Repeatable Read` gets a snapshot, each individual transaction will be successful, resulting in a large net loss to the bank. It is worthy of note here, that while a lot of work has been done to optimize the higher transaction levels, it is still true that the higher transaction levels come with some performance overhead. Whether or not the higher consistency is worth the performance cost, is something that should be evaluated on a case by case basis, using benchmarks on your data and operations. 

#### 2.2.1 Transactions in MongoDb 

MongoDB does not support transactions out of the box. It does however allow, configuration of its `Write Concern`, the "guarantee that MongoDB provides when reporting on the success of a write operation. The default is `Acknowledged`, which guarantees that the write operation has reached the database, but *does not* guarantee that the data has actually been written to disk. Other options are: 

`Journaled`: the write request has been written to MongoDB's Journal (A queue of operations that has not yet been persisted to the disk proper) 

`Majority`: The write has propagated to the majority of nodes, and been acknowledged by them. A lengthy discussion of the various failures of MongoDB's write concerns can be found here:
https://aphyr.com/posts/322-call-me-maybe-mongodb-stale-reads 

The short version is that while `Majority` does ensure consistency in the absence of network partitions, network partitioning may result in inconsistent data, and/or data loss, even within the confines of a single document. It is also true that MongoDB does provide an `$isolated` operator, which, while enforcing consistency by way of a write lock, also prevents all sharding, and does not actually guarantee atomicity as an error during the write operation does not roll back the entire "transaction". MongoDB also does provide a loose guideline for implementing a two phase commit:

https://docs.mongodb.org/manual/tutorial/perform-two-phase-commits/ 

Loosely speaking this will require you to:
Create a transaction with all of your queued changes.
Execute each change, sequentially, tracking the succeeded changes.
If any change fails, roll back every change already made, and cancel the transaction.
Mark the transaction done.

Keep in mind that this means you need to be able to encode an inverse, or rollback operation for every operation, and that this is a very hard problem to get right, a full solution for which is out of scope of this article.

#### 2.2.2 Transactions in WiredTiger

Wiredtiger purports to provide ACID transactions (I do not have sufficient computing power to put this to the test, and unfortunately third party tests have yet to occur). Taking the documentation at its word, it provides a maximum level of Snapshot Isolation, equivalent to Postgres' Read Committed. 

Winner: Postgres. By knockout. Although once WiredTiger is ready for primetime, this gap will narrow somewhat.

### 2.3 Performance and Denormalized Data 

#### 2.3.1 Denormalized data in Postgres. 

Postgres supports 4 column types for storing denormalized data. 

`hstore`: this is for storing flat key - value pairs. It is largely only there for legacy use. 

`json`: this stores json blobs converted to a string. It does run some validations to ensure well formed json, and provide convenience operators for access, but that's about it. It does not really provide indexing on This is also largely only for legacy use. 

`jsonb`: This stores json as binary, and displays it as json, not as a single text value, and allows indexing into arbitrary attributes for speed of lookup. It does however still require a full update to write to, although there are functions coming out in 9.5 that will allow updates to nested paths more easily: https://michael.otacoo.com/postgresql-2/postgres-9-5-feature-highlight-new-jsonb-functions/ 

'array`: This stores an array of some other datatype (text, number, whatever). To find out more about the operators available on all of these, the documentation is a great place to go: https://www.postgresql.org/docs/9.4/static/functions-json.html 

#### 2.3.2 Denormalized data in Mongo 

Storing Json is literally the only thing Mongo does. Mongo stores its data in a binary format called BSON, which is (roughly) just a binary representation of a superset of JSON. The reason for the roughly quantifier is the lack of a number type, while the reason for superset is the support for direct binary data. A pretty good place to learn more is the documentation: http://docs.mongodb.org/manual/reference/mongodb-extended-json/ 

#### 2.3.3 BSON VS JsonB 

So how do our contenders perform? The first thing an astute reader will notice, is the similarity between jsonb and BSON, both are a json structure stored as binary internally. SHow do they differ? The first point of difference is that JsonB will output fully standards compliant Json, as described by http://rfc7159.net/rfc7159, while BSON has never gone through any formal peer review. This is, however a double edged sword. For example, JsonB does not support a native binary type unlike BSON, nor, more pertinently a date type. Whether you require more complex types to be explicitly type checked in your Json Store, or value the inherent strict standards compliance of a peer review process, is honestly a decision only you can make based on the business problems you are attempting to solve.

#### 2.3.4 Performance Comparison 

As demonstrated here: Postgres is faster, uses less memory on disk, and is all around more performant for JSON storage and reads then Mongo. https://www.enterprisedb.com/postgres-plus-edb-blog/marc-linster/postgres-outperforms-mongodb-and-ushers-new-developer-reality 

Winner: Postgres. By technical Victory. Postgres is faster, less memory intensive, and more standards compliant then MongoDB. However, if you require some of the intrinsic type checking of BSON, and this type checking must be done in a denormalized manner, rather then by table columns, or if your need to do complex access updates on a JSON attribute, Mongo ekes out a victory. 
Once Postgres 9.5 ships this becomes a knockout. 

### 2.4 Complex Model Relations, Access Patterns, and Normalized Data 

#### 2.4.1 Normalized Data in Postgres 

This is more or less what postgres does. It allows you to encode data relationships into tables, using foreign keys to encode relationships between tables. It allows you to join between tables to pull in data from across table boundaries. For example: 
```PLpgSQL 
CREATE TABLE USERS( id SERIAL PRIMARY KEY, organization_id INTEGER, name TEXT ); 
CREATE TABLE ORGANIZATIONS( id SERIAL PRIMARY KEY, name TEXT ); 
``` 
Now to select we can either: 
```PLpgSQL 
SELECT USERS.* FROM USERS, ORGANIZATIONS WHERE USERS.organization_id = ORGANIZATIONS.id AND ORGANIZATIONS.name LIKE '%bar%'; 
``` 
Or 
```PLpgSQL 
SELECT USERS.* FROM USERS INNER JOIN ORGANIZATIONS ON USERS.organization_id = ORGANIZATIONS.id WHERE ORGANIZATIONS.name LIKE '%bar%'; 
```
http://sqlfiddle.com/#!15/b5f9e/2 

#### 2.4.2 Normalized Data in MongoDB 
Loosely speaking Mongo collections map to Postgres tables, while Mongo Documents map to Postgres Rows. It is important to note that MongoDB does not support Joins, forcing you to query for nested relationships directly if you choose to store either a direct key id or a DBRef, or letting you fetch directly from the nested object. While storing a nested object like so: ```Javascript 
db.createCollection('users'); 
db.users.insert({name: 'jack', organization: {name: 'foo corp'}}); 
``` 
Is a trivial solution, it is also denormalized, and from a business logic point of view, presumes that organization has no life, independently of user. What if this isn't true, or you would like to have normalized data? MongoDB allows two ways to achieve this: 
1) DBRefs allow you to embed direct references to other documents in your documents. However, this will force additional queries to be run every time to fetch your referenced documents, and should (as per MongoDB's documentation) be avoided when possible 
2) 
```Javascript 
db.createCollection('users'); 
db.createCollection('organizations'); 
db.organizations.insert({name: 'foo corp'}); 
db.organizations.insert({name: 'bar corp'}); 
var foo = db.organizations.find({name: 'foo corp'}); 
db.users.insert({name: "Jack", organization_id: foo}); 
``` 
You can then use application level logic to extract the organization_id, fetch that organization separately, and join the data in the application. Note that, this bypasses any transaction logic you have built or use, unless your transaction logic is handled at the application level. Winner: Postgres By knockout. 

### 2.5 Scaling 

There are fundamentally two types of scaling, horizontal scale, and vertical scale. Vertical scaling loosely means adding resources to a given machine. More RAM, more CPU cores. Horizontal Scale means multiple machines running your database. 

#### 2.5.1 Scaling Postgres 
As long as you can maintain Vertical Scale, Postgres scaling is trivial. You add more power to your machine, bump up the resources allocated to Postgres, and you're off to the races. But sooner or later this will hit a ceiling. Scaling Postgres Horizontally is significantly harder, but doable. There are several valid strategies to achieve this. At the core these are replication for reads (a master machine that allows writes, and multiple read only machines), and sharding. Sharding is a complex topic with multiple solutions, from application level load balancing, down to database level logic to store the shardid as part of the primary key as done at instagram: http://instagram-engineering.tumblr.com/post/10853187575/sharding-ids-at-instagram A detailed discussion of the approaches is out of scope of this article.

#### 2.5.2 Scaling Mongo 

MongoDB Supports Sharding at the technology level. When sharded, collections are partitioned by a `shard key`. Mongo's `query routers` can then identify the right shard to read from. A great resource to achieve good sharding, complete with best practices for balancing shard sizes, can be found here: http://docs.mongodb.org/master/MongoDB-sharding-guide.pdf 

Winner: MongoDB. Technical Victory due to native sharding support and ease. 

### 2.6 Rapid Prototyping 

So, you have investors breathing down your neck, and you owe a prototype yesterday. What technology do you choose for your data store? While Postgres seems to be the clear victor above, there are a few advantages to using Mongo: 
1) As the store is schema-less, as your requirements rapidly change you do not need to continuously write migrations 
2) You do not need to think through your data-model, ensuring normalization. 
3) You do not need to write SQL, as The query language is JSON like, and will feel very familiar to anybody with Javascript experience. 
4) It is probably fair to say that at this early juncture a lot of your data is of suboptimal importance, and your organization can survive its loss or corruption, thus the strong guarantees provided by Postgres are not necessary.
The downside: All data is equally likely to be lost. If your organization deals with enterprise customers, or handles financial data, MongoDB is very simply not an option (http://hackingdistributed.com/2014/04/06/another-one-bites-the-dust-flexcoin/). Additionally, while it is true that MongoDB is easier to scale down the road, Postgres is also scalable (if its good enough for Instagram....). 

Victor: MongoDB, Technical Victory. Assuming you do not already have postgres and/or database expertise, MongoDB's simpler query interface and lack of requirements for schema migration/maintenance make it easier to rapidly prototype in. Just be aware that unless you fit a small set of niche use cases where individual, small scale dataloss is truly irrelevant (running large scale analytics on normally distributed datasets), you will eventually have to throw your database away and rewrite swathes of your application. 

## 3. Summary 
Postgres comes out the clear victor of this fight. There are valid use cases for MongoDB such as reporting on large datasets of normally distributed data, and storing TRULY denormalized data, data where relationships are mostly non existent, and documents are large, mostly unstructured, and with little to no overlap. However as a general purpose database, Postgres is clearly the dominant fighter in this arena, and if some denormalized data is required, like say a set of optional parameters on a user (eye color, height, weight, hair color), Postgres' JSONB column is more than sufficient.
