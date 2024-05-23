# Designing Data Intensive Applications

A more in depth reading and notes from DDIA

Contents
========

 * [Chapter 1](#chapter-1)
    * [Reliability](#Reliability)
    * [Scalability](#Scalability)
    * [Maintainability](#Maintainability)
 * [Chapter 2](#chapter-2)
    * [Data models and Query languages](#Data-models-and-Query-languages)
    * [The relational model](#The-relational-model)
    * [Document model](#Document-characteristics)
    * [MapReduce](#mapreduce-querying)
    * [Graph databases](#graph-databases)
* [Chapter 3](#chapter-3---storage-and-retrieval)
    * [SSTables and LSM trees](#sstables-and-lsm-trees)
    * [BTrees](#b-trees)
    * [Other types of indexes](#other-types-of-indexes)
    * [OLAP vs OLTP](#olap-vs-oltp)
    * [Column oriented](#column-oriented-storage)


# Chapter 1

## Reliability
- the system should continue to work correctly when dealing with advertity (to the most of its ability)
    - ties strongly into consistency, i.e. failures along durign a request's execution should not leave the system in an inconsistent state
- **fault tolerance** refers to one of the components of the system not behaving as expected. 
    - we cannot reduce the chance of failures to zero (human/hardware/software errors), thus it is only a matter of time until a failure occurs. 
    - in effect, any reliable system should expect these failures to happen and work on mitigating them as much as possible

### Hardware failures

- since most services are deployed in a hosted manner (AWS, Azure), there are no strong guarantees of VM availability
    - on the contrary, machines becoming unavailable is a routine occurrence that should be planned for
- the cloud providers themselves are responsible for ensuring reliabiity at this level usually, e.g. by spinning up a new VM when one dies

- the network connection between any two nodes can fail at any point
    - futhermore, we cannot know for a fact the nature of the failure. A timeout for example can mean 
        - our message never reached the destination, so its recepient is not aware of it
        - it reached its recipient, which performed some actions in reaction to it (e.g. a write operation), but we failed to receive their response

### Software errors

- as opposed to hardware failures, which tend to be localized (e.g. hard drive failure over distance is not related), software bugs have the potential to take out the entire service
- usually surface when the underlying assumptions stop being true, e.g. that a service we depend on will continue to return successfully
- measuring the system behavior in production is essential; it can also help validate our assumptions, e.g. an end-to-end operation results in what we expect it to
    - this way, the system can constantly check itself and report any inconsistencies

### Human errors

- the most prevalent source of issues; poor configuration is more likely to be the cause of an outage
- several ways to minimize the likelyhood:
    - design the API / library such that it's easy to do the right thing and hard to do the wrong one
    - setting up clear monitoring, such as performance metrics and error rates
    - can give an early warning for failures, as well as allow for diagnosing the issue


## Scalability
- the system should be able to accomodate growth: both in the total amount of data it stores, as well as the incoming rate. 
- we should also be able to accomodate more complex data coming in as the requirements evolve over time

## Maintainability
- the various (and changing) set of people working on the system should be able to do so *productively*, e.g. 


# Chapter 2 

## Data models and Query languages

Data models reflect the way we think about the problem we are solving. Once we chose the right models and abstractions for our problem, the solution is usually obvious.

There are several layers of models, stacked on top of each other:

1. The internal models of our application: objects, data structures and APIs. These are specific to the application being developed.
    - an example: a payment request and transaction. These are domain specific.
2. When persisting these, we need to map them to a representation the storage layer would understand, e.g. tables, json documents etc.
    - an example: mapping a payment order to a row in a dynamo table
3. The storage engine represents its objects in memory / on the disk. Based on the chosen implementation, the entire layer is efficient for some operations, while not idea for others
    - an example: doing a many-to-many join is less than ideal for dynamo, since we need to make several requests to get the entire model

Each layer hides the complexity below it and exposes a simple abstraction to be used by the callers. Examples: a paygate API that is a facade for the underlying operations.

## Options for reprensenting our models

1. In a traditional SQL fashion, we can represent our models in separate tables, with FKs referencing other entries. 
    - Example: payment orders table and payments table, with the payment row having a FK to the payment order Id
    - relatively straighforward if the relations are 1-1 or 1-many, e.g. a **tree structure**.

2. Storing the entire document as a JSON in a column. Tipically we can't search / index within this document, but we let the application interpret its content.
    - storing the object as a JSON makes the relationship between the nodes explicit, e.g. payments live under a payment order

Storing IDs vs storing values:

- IDs never need to change (as opposed to names etc), as they are not meaningful to humans. Thus they are more flexible.
- referencing an ID removes duplication. Storing the text directly would require duplicating it in every record that uses it.
- storing the ID requires a Join to resolve it to the actual textual value.
    - for relational DBs, this is straightforward and supported by the engines themselves
    - for non-relational, it usually involves making multiple queries to the DB

Example: storing user information inside the payment order vs storing the user ID
- if we store an ID for the product, this can be resolved when querying (by also making a query to retrive the user and their details based on its ID)
    - this allows the information to be dynamic, e.g. a user can change their name
- if we store the information directly, it removes the need for another query
    - however, we'd need to update all the duplicated information should something change for that user

## The relational model

- all the data is laid out in the open - a table is a collection of rows with no other special meaning
- one can query a subset or all the records in a table
- the query optimizer decides how to execute the query, **and which indexes to use**. Adding a new index does not involve code changes, it will instead be chosen automatically if it would help with a particular query's execution.

### Relational charateristics:

- better support for joins 
- better 1-n / n-n relationship modelling
- you can refer to any object directly
- validates the schema on write; if no match, the DB will prevent the write (statictly typed checking)
- schemas are a useful mechanism when we expect all records to follow the same structure. It can be used to document and enforce this.

### Document characteristics:

- ideal where data comes in self-contained documents and relations between documents are rare
- easier to map to actual objects being used by the application, as we can deserialize the entire tree into an in-memory model right away
- if using nested objects, we cannot reference them directly. E.g. if all payments for an order live under the same entry for the Payment Order, we wouldn't be able to retrieve one of them without knowing their parent
- "validates" the schema on read, but deserializing it into an object with an implicit structure (dynamic type checking)
- modifying the schema is as simple as writing in the new format
    - the burden of keeping both versions in check falls to the application code
- data locality:
    - the entire document it stored in a contiguous block on the disk
    - if we often need to access large parts of it at the same time, reading will be faster due to *storage locality*
    - as opposed to this, multiple index lookups will probably need to read multiple disk locations, thus be slower (even if it's all part of the same query)
    - if we only use a small part of the document, the DB will still load the entire value. which is wasteful
    - keeping the documents small is ideal, as well as avoiding rewrites (the entire value has to be overwritten)


## Query languages

Two main approaches: an imperative vs a declarative description.

### Imperative

- defines a list of operations instructions to be executed, in a particular order
- the query engine doesn't know if the caller relies on the order of the results being returned, so it cannot alter it for optimizations
- is a poor candidate for parallelization, as we need to guarantee the order in which the operations are executed
- we can ofc shard the data and aggregate the results if possible, but it leads to more code
- Javascript as an example

Example:

```
for (i in 1..n) {
    list.add(n)
}
```

### Declarative

- defines conditions the patterns has to meet
- delegates all other implementation details to the query engine
- offers fewer guarantees, thus is more generic (and flexible)
- css as an example

Example:

```
list.filter { it.name == "Europe" } 
```

CSS example
- defines the pattern which an element has to meet in order to apply the style
- all <p> elements where a selected <li> element is its parent

```
li.selected > p {
    background-color: blue;
}
```

## MapReduce querying

0. A filter is specified declaratively
    - this allows the query engine to select which items are relevant for the operation
1. The map function is imperative and it is applied once per each element
    - it returns a key and a value
2. The pairs are grouped by key. For all key-value pairs with the same key, the reduce function is called once with the entire set
    - the reduce function returns a single value for the entire group

The map and reduce functions must be pure:
- no reading the database
- must only use the data that is passed into them
- no side effects (as we they might be run multiple times in cases of failures)
- they can use libraries, do computations etc

For a list of documents having three properties: family (for querying), timestamp (for grouping) and numAnimals (for reducing):


```
db.observations.mapReduce(
    function map() {
        var year = this.timestamp.getFullYear()
        emit(year, this.numAnimals)
    }
    function reduce(key, values) {
        return Array.sum(values);
    }
    {
        query: { family: "Sharks" },
        out: "sharkReport"
    }
)
```

## Graph databases

When the relations between objects are mostly N to N, and the types of relations are varied.

Modeling the data as a graph allows us to leverage algorithms designed for graph traversal.

**Ideal for scenarios where anything can be related to anything, i.e. plenty of N-to-N relations**

![GraphDB](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/graph_db.png)

Components:
- a set of vertices, containing all the types of objects in our model
    - each model is its own document, with a list of properties
- the relations between them
    - this can be modelled as a (pair of ) directed, or undirected edge. E.g: 
        - two people are both married to each other
        - an actor can have a "playedIn" edge to a movie, while the movie has a "hasActor" edge to the same actor
- each edge has the following:
    - a unique ID
    - a start vertex ID
    - an end vertex ID
    - a label to describe the type of relation, e.g. played-in 
    - ***a collection of properties***

It's simple to get the list of all edges either incoming or outgoing from a vertex.
- this allows us to traverse the graph starting at any vertex, in both directions
- using different labels keeps the model clean, while allowing for multiple types of relations to be described

Example for creating the graph: (is it worth it?)

Example for querying:

```
MATCH
    (person) -[BORN_IN] -> () -[WITHIN*0..] -> (us:Location { name:'United States' })
    (person) -[LIVES_IN] -> () -[WITHIN*0..] -> (eu:Location { name:'Europe' })
```

- this query finds any vertex that matches two conditions:
    - it has a BORN_IN edge to another vertex; from this vertex, there exists a path of 0 or more WITHIN edges to a vertex of type Location, that has a name of 'United States'
    - it has a LIVES_IN edge to another vertex; from here there exists a path of 0 or more WITHIN edges to a vertex of type Location, that has a property called 'name' with a value of 'Europe'

Multiple ways this query might get executed:
- one can start with all the people and traverse the graph until it matches (or not) the criteria
- we can also start with all the ending vertices and move towards other locations, and eventually people.
- given the language is declarative, we don't care about these optimizations; the query engine takes care of chosing the best approach

# Chapter 3 - Storage and retrieval

The previous chapter covered the way that we shape the data and store / query it in the database. Now we'll look at how this is reprensented internally in the storage engine. 

## Data structures used by the databases

- Two main categories of storage engines:  
    - log-oriented
    - page-oriented

Designing a simple key-value store:
- we use an append-only log, storing both the key and the value
- very efficient for writes, as we just append at the end of the file
- very inefficient for reads, as we need to parse the entire file to get a value (O(n))

An **index** is an additional structure derived from the main body of information. 
- adding or removing one has no impact on the data itself
- by their nature, they will always slow down writes (as the index needs to also be kept updated)
- well designed indexes will speed up reads. However, which indexes to chose is up the the application developer
- given their drawback, most engines don't add indexes by default

### Hash indexes

- using our previous example of an append-only log file, we want to use an index to improve the read speed
- we can store an in-memory structure that ***maps each key to its latest offset in the file***
    - on writes, we can do an O(1) lookup into the index and update it
    - on reads, we do an O(1) lookup to find the offset, and another O(1) to read the value
    - the main drawback is the fact that the entire key space has to fit in memory
    - **range queries are not efficient, as the keys ar not adjacent**
- this is very efficient for scenarios where the values are updated quickly, but the key space is contained
    - an example is video counters, where the number of actions (views) is much larger than the number of keys (videos)
    - Riak offers a similar implementation

Problems arise when the log file gets too large, as it cannot grow infinitely.
Due to the old values not being relevant anymore (as we only care about the current one), we need a way to reclaim the space.

1. When the log file grows to a certain size, we stop accepting new writes to it and open a new one (and a corresponding hash index)
2. When a read request comes in, we first check the current file's hash index. If the key is present, return; otherwise, we look backwards through the older log files
3. To reclaim data, we have a background job that:
    - reads the old log files and only keeps the latest value for each key (sweep)
    - merges the smaller segments from the above step across multiple files (compact)
    - while the task runs, keep serving reads from the old files
    - once it's done, point the reads to the newly compacted files and delete the verbose ones

Other considerents

- deleting records:
    - we need a special marker that expresses the value is deleted (called tombstone)
    - when this is present explicitly, just discard the value
    - when doing the sweep, we can just not copy it

- crash recovery:
    - the hash maps will be lost on a reboot;
    - however, we can save them to disk. For closed files, they will never change, while for in-use files we can keep flushing them regularly as backup points

- partially written records:
    - we can use a checksum for detection
    - how? do we just discard the latest entry in the file?

### SSTables and LSM-Trees

To overcome the need to keep all the keys in memory, as well as the inefficiency for range queries, we can change how the index is built.

We'll be using an additional in-memory structure that allows us to keep the keys in order (e.g. a red-black tree). This is the *memtable*

For writing:

1. A request comes in and is appended to the WAL, for recovery purposes
2. We keep an ordered in-memory structure with the current set of key-values (memtable)
    - appending and reading from it should be simple and fast
3. When the in-memory structure gets too large, we flush it to disk as an SSTable (Sorted String Table)
    - this is efficient, as the tree already has the keys sorted
    - while we write it to the disk, additional writes go to a new memtable
4. For each SSTable, we continue to require an in-memory structure mapping the keys to the offsets
    - however, since the keys are now sorted in the SSTable, we can keep a *sparse* structure - we only store a subset of the key / offset pairs

For compacting:

1. We take a list of existing SSTables flushed from the memtable (or previous compactions)
2. Since all of them are in order, we can use merge-sort to merge them into a single SSTable
    - we discard equal values from earlier segments, as those got overwritten
    - for the new SSTable, we build the new sparse in-memory index
    - the ranges between the values in the index can be compressed before writing it to the disk
    - this is fine, as we'll read the entire range anyway

For reading:

0. We use a Bloom filter to determine if the DB might contain the key; if not, return
1. We check the memtable. If it contains the key, we just return the value
2. If the memtable does not contain the key, we need to find it:
    - we process the SSTables from the newest to the oldest to try and find it
    - for each one, read the in-memory index and determine a candidate range / compressed block
    - if any viable range, read the block and check for the value
    - repeat until found, or we run out of segments

The LSM-Tree is the entire grouping of SSTables, indexes and the memtable.

### B-Trees

Another approach is to breakdown the database into **fixed** size blocks (usually 4KB) and read / write one page at a time.

![B-Tree insertion](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/btree.png)

Each page is identified using a location on the disk, which allows us to build a tree of pointers, each referencing a page. 
- each child is responsible for a continuous range of keys, which are stored in order
- the keys between the references define the boundaries of the intervals
- the **branching factor** (how many intervals a node contains) determines the depth of the tree. A factor of 500 with 4KB pages can store more than enough information.

When reading:
- start from the root
- use binary search to find the child that might contain our value or interval
    - repeat until we reach a leaf node (which contains actual values) and check the value is there

When writing:
- for updates:
    - find the leaf node and change the value; all references to that page remain valid
- for inserts:
    - find the leaf node and add the new value
    - if it doesn't fit, split the page into two halves, and update the parent with two intervals instead of one
        - should the parent be full, bubble up the change (this is in effect an insertion into the parent, as one interval turns into two)
- page update:
    - we can write the new value of the page to a new location and update the reference in the parent, instead of overwriting in place
    - a page might contain pointers to its siblings, to allow for sequential scanning

### LSM vs B-trees

LSM wins:

- we write just the information that is needed, as opposed to an entire page
    - NOTE: appending to the WAL does not really result in a write to the disk immediately. 
- we write compact SSTables sequentially, as opposed to random pages at random addresses
    - might benefit from data locality, i.e. if the SSTables are next to each other, the might already be loaded in the CPU cache optimistically
- LSM trees don't leave space unoccupied, as opposed to B-Trees, which have a fixed-size page
    - this page is mostly empty when initially created

B-Tree wins:

- no background processes to interfere with the DB operations
    - for LSM, the incoming DB writes and the compacting are competing for the same resources
    - more predictable throughput, **especially at high percentiles**
- the key exists in a single place
    - this makes it easier to lightly lock for transactional purposes

## Other types of indexes

A B-Tree/LSM tree offers indexing on a single key, similarly to a primary key. Thus, each row/document/vertex is uniquely identified by the key.

Secondary indexes are similar, except they might have more than one value corresponding to the same key.
    - we can solve this by either having a list of IDs associated with the value of the secondary index
    - or by making each entry unique, via adding a unique identifier to it

### Storing values within the index

The value we store can be:
- the actual value
- a pointer to the actual value's location; In this case, the rows are stored in a **heap file**.
    - this approach helps prevent duplication, as each seconday index can simply reference the relevant row in the heap file
    - when updating a value:
        - if it fits in the same location, no further changes
        - otherwise, we need to update its references too, or to leave a **forwarding pointer** 

A **clustered index** is an index that has the information in-place. This can be the primary index of the table, e.g. our B-Tree. 
    - other secondary indexes can simply reference the row IDs.
    - an example:
        - the B-Tree uses userId as the key, with each value being unique
        - we build a secondary index for the last name
            - this can result in a new B-Tree, partitioning the keys alphabetically
            - the leafs (actual values) can just store a list of userIds corresponding to that last name
            - thus we can obtain a list of values for the secondary index in log (depth) (N)
    - another approach is to partially store some information in the seconday index, so we wouldn't have to perform another lookup against a set of primary keys
        - for example we can also store the first name values in the example above
        - this has the downside of having mutliple copies of the information, so ensuring transactional guarantees and preventing inconsistencies becomes more difficult

### Multi-column indexes

- The current structures allow us to query on a single dimension, that of the primary key. It's easy to get all users with an ID between N and M, but not those that also have a birthday in June.

A **concatenated index** is simply a concatenation of the values from multiple columns / attributes. E.g. mapping {lastName + firstName} to a phone number: easy to query by the combination or by the lastName, but not so much by firstName. 

R-Trees used for this. TODO - add more info here - https://www.bartoszsypytkowski.com/r-tree/ 

### Fuzzy searches

- a regular B-tree or LSM tree allow for querying only for a precise key
- a misspelled one might not be close to the actual key looked up (e.g. first char is different)
- some engines allow for querying for keys within a certain edit distance than the key (number of different chars)
- there are dedicate algorithms that allow for searching within an edit distance, based on the in-memory (sparse?) index
    - we can transform it into a preffix tree and not just look for a direct parent-child link, but also take into account the current char might be misspelled
    - Levensthein distance?


### Keeping all the data in memory (a-la Redis)

- due to the erasure on power loss, it is considered to be volatile
- most are suited for keeping not-critical data, but we can also keep a WAL to persist the information and rebuild the in-memory representation if needed
- another disadvantage is that we can't easily inspect the contents of the DB; a DB that writes its data to disk allow for backups and inspections
- an advantage is enabling new data structures that would be difficult to implement with persistent storage, e.g. a priority queue


## OLAP vs OLTP

- two main use cases for a data storage:
    - queries for real-time usage (OnLine **Transaction** Processing):
        - Reads: small number of records per query, retrieved by their keys
        - Writes: random-access, should be low-latency
        - User: a user via an interaction with the service
        - Data version: uses the latest version of the data
        - Size: GB to TB of information
    - queries for analytics (OnLine **Analytics** Processing)
        - Reads: large number of records per query
        - Writes: either in batches (ETL), or via event streaming
        - User: data analyst
        - Data version: all of it, has a history of events that happened
        - Size: TB to PB

Why separate:
- OLTPs are critical to the business operations
- each team usually manages their own service / data
- long running queries against the OLTP can affect their performance
    - resources are shared
    - transactions are also affected via locking

### Data warehouse

- a system where the a copy of the information from various sources is loaded
- information is either loaded:
    - in a continous stream (via event sourcing)
    - in batches, via ETL - Extract, Transform, Load
- can be optimized for different access patterns than the OLTP

The usual schema for a data warehouse is a star / snowflake:
- a fact table is the core of the model and stores most of the information
    - each row represents an event (e.g. a payment order was created, paid, etc)
    - each event is captured separately
        - allows for a historical view 
        - however, the table will grow at a faster pace
- the fact table references other tables, or *dimensions*

![Star schema](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/star_schema.png)

- the tables often have a large amount of columns (e.g. hundreds), while queries only care about a handful of them

## Column-oriented storage

- in most OLTP DBs, storage is laid out in a row-fashion (all the information in a row is located in the same place)
- in a document DB, the entire document is stored in one place (contiguous sequence of bits)

- when querying, the entire row / document need to be loaded into memory, which is inefficient

Alternatively, we can keep the information from the same column together. 
- each file is a list of values, corresponding to the values of all the records for that column
- the ordering is consistent across the column files, i.e. the 3rd element in each of them corresponds to the same row
- this way we can reconstruct a row whenever needed

Advantages:

- When querying for a subset of column, only those need to be loaded from the disk
- data in one column might have similar values for different rows, thus a good candidate for compression

### Column compression

- Based on the idea that a column will have a (small) finite number of possible values.
- for each possible value, we compute a bitmap (array of bits) based on the column values:
    - if the n-th row's value matches the one we're building the bitmap for, the n-th value in the boolean array will be 1
    - 0 otherwise
    - when doing a query for product_sk in (23, 35, 42), we can look up 23's, 35's and 42's bitmaps 
        - we compute a bitwise OR on the bitmaps
        - the result's positions for 1s will be the row IDs that match the query
    - when doing a query for product_sk = 3 and store_sk = 7
        - we apply 3's bitmap on the product_sk column
        - we apply 7's bitmap on the store_sk column
        - apply a bitwise AND between the results
        - the result's positions for 1s will be the row IDs

Additionally, we can also perform run-length encoding on the bitmaps:
- an array of 0 0 0 0 0 1 0 0 0 0 can become 5,1 (5 zeroes, 1 one, rest zeroes). 
- this can improve the compression even further

To sum it up, we're relying on pre-computed values (the bitmaps) in order to answer queries faster when needed. 
- these bitmaps need to be updated when new information is being written

![Column compression](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/column_compression.png)


### Vectorized processing

TODO - add more information here

https://www.infoq.com/articles/columnar-databases-and-vectorization/

- when scanning over millions of rows, data needs to go from disk -> memory -> CPU
- the operations for bitmaps can be executed directly on the compressed data (how?)

### Sort order

- we can insert in the order in which the write calls come in, similarly to a log (by appending to the end of the column files)
- alternatively, we can keep them in order (similarly to the SSTables) and get fater lookups

- Based on the types of queries we want to answer, we can chose a different column for which to sort
- scanning over a sorted column is obviously much faster, as we can identify the relevant rows directly
- since adjacent values will be adjacent in the file, we also take advantage of data locality

- we can define multiple sort columns, to be applied as GroupBy.ThenBy.ThenBy etc.
    - the locality effect is also applied by induction. E.g. groupBy product_sk, then by store_sk
        - within the group of relevant rows identified for product_sk, they will be ordered by store_sk so we can filter more easily
    - having the same values next to each other also helps with compression
        - this becomes less and less relevant the deeper we go, as the values will not be in any order relative to the table

Different ordering:
- since we need to keep multiple copies of the data, we can also keep them sorted using different columns
- this is similar to having mutliple secondary indexes in a relational DB

Writing:
- all the optimizations above are helping with reading, but are making writes more difficult
- when the data is compressed, updating it in place is inefficient
    - inserting a row would mean rewriting **all** the column files, as they would need to be kept in order

- instead, we use a memtable as for LSM trees:
    - writes come in, we update the in-memory structure
    - when it's full, it is flushed and merged with the existing compressed files
- queries need to look at both disk data and the memtable, but the query engine abstracts this

### Data cubes

- similarly to how we precomputed the bitmaps for values in the same column, we can precompute query results
- a **materialized view** is a persisted table, caching query results for particular dimensions:

![Materialized view](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/materialized_view.png)

This has two downsides:
- when the value of any column changes, this needs to be updated (since it's a copy of the data); **thus writes are slower**
    - if the ratio of reads to writes is worth it, this will however overall speedup the querying
- we cannot get more granular data, e.g. which sales came from which products


# Chapter 4 - Encoding and schema evolution

Two main concerns:
- how do we store the data on the disk in a more efficient format?
- how does the application stay forward / backwards compatible, given the data format?

A rolling update will result in:
- new versions reding old data / getting old requests
- old versions reading new data / getting new requests

## Formats for encoding

Two types of representations:
- in-memory: objects and their pointers, optimized for access
- serialized data - self contained sequence of bytes

### JSON

- verbose compared to binary formats
- no option of specifying the type, e.g. when encoding numbers (int vs float, precision)
- easily readable => a good choice for inter-organization communication (i.e. people boundary)

A base example:

```
{
    "userName": "Martin",
    "favoriteNumber": 1337,
    "interests": [ "daydreaming", "hacking"]
}
```

### Binary formats

- the underlying idea is that we'll have multiple records following the same structure
- encoding the schema in each of them is not needed; if both the reader and the writer agree on the structure, a simple stream of bytes can be sufficient (and much more compact)

- the schemas between the writer and the reader must be compatible. This applies to both:
    - JSON: a required field by the reader must be set by the writer
    - binary formats - byte offsets might mess up the entire deserialization.

- many applications have their proprietary data formats / protocols, e.g. a network protocol for databases

### Avro

- separates the schema from the data
- when ecoding, we're using a *writer's schema*
- when decoding, we're using a *reader's schema*
- the library takes care of mapping fields from one to the other, as long as they are compatible
    - the order of the fields might change
    - a field might be renamed, but the reder's schema must contain an alias to map it
    - if a reader expects a field, the deserialization will provide a default value
    - extra fields from the writer are discarded when reading

![Avro encoding](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/avro.png)

Due to size considerents, we don't want to include the schema for each record, as this would negate the entire benefit.
This is a good solution for cases when we have one encoding and multiple records:

1. File with lots of records
    - we include the schema once at the start of the file, and all records are expected to follow it
    - con: can't vary the schema within the file
2. Database with individual records:
    - each record might be written with a different schema
    - add a tag to each row to specify which schema version was used
    - have a separate location that maps the schema version to the actual schema
3. Sending records over the network:
    - the schema can be negociated on conection setup (e.g. accepts)
    - we assume the schema will not change during the lifetime of the connection (which should hold, we change the schema on redeployments / restarts)

## Data flow

- the process of sending information to a process with which we don't share any memory (same machine, remote etc)

### Through databases

### Through service calls

### Through message brokers