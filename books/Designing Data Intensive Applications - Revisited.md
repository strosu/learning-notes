# Designing Data Intensive Applications

A more in depth reading and notes from DDIA. All images taken from the book, unless specified otherwise.

- [Designing Data Intensive Applications](#designing-data-intensive-applications)
- [Chapter 1](#chapter-1)
  - [Reliability](#reliability)
    - [Hardware failures](#hardware-failures)
    - [Software errors](#software-errors)
    - [Human errors](#human-errors)
  - [Scalability](#scalability)
  - [Maintainability](#maintainability)
- [Chapter 2](#chapter-2)
  - [Data models and Query languages](#data-models-and-query-languages)
  - [Options for reprensenting our models](#options-for-reprensenting-our-models)
  - [The relational model](#the-relational-model)
    - [Relational charateristics:](#relational-charateristics)
    - [Document characteristics:](#document-characteristics)
  - [Query languages](#query-languages)
    - [Imperative](#imperative)
    - [Declarative](#declarative)
  - [MapReduce querying](#mapreduce-querying)
  - [Graph databases](#graph-databases)
- [Chapter 3 - Storage and retrieval](#chapter-3---storage-and-retrieval)
  - [Data structures used by the databases](#data-structures-used-by-the-databases)
    - [Hash indexes](#hash-indexes)
    - [SSTables and LSM-Trees](#sstables-and-lsm-trees)
    - [B-Trees](#b-trees)
    - [LSM vs B-trees](#lsm-vs-b-trees)
  - [Other types of indexes](#other-types-of-indexes)
    - [Storing values within the index](#storing-values-within-the-index)
    - [Multi-column indexes](#multi-column-indexes)
    - [Fuzzy searches](#fuzzy-searches)
    - [Keeping all the data in memory (a-la Redis)](#keeping-all-the-data-in-memory-a-la-redis)
  - [OLAP vs OLTP](#olap-vs-oltp)
    - [Data warehouse](#data-warehouse)
  - [Column-oriented storage](#column-oriented-storage)
    - [Column compression](#column-compression)
    - [Vectorized processing](#vectorized-processing)
    - [Sort order](#sort-order)
    - [Data cubes](#data-cubes)
- [Chapter 4 - Encoding and schema evolution](#chapter-4---encoding-and-schema-evolution)
  - [Formats for encoding](#formats-for-encoding)
    - [JSON](#json)
    - [Binary formats](#binary-formats)
    - [Avro](#avro)
  - [Data flow](#data-flow)
    - [1. Through databases](#1-through-databases)
    - [2. Through service calls](#2-through-service-calls)
    - [REST](#rest)
    - [RPC](#rpc)
    - [3. Through message brokers](#3-through-message-brokers)
- [Chapter 5 - Replication](#chapter-5---replication)
  - [Synchronous replication](#synchronous-replication)
  - [Async replication](#async-replication)
  - [How does the information get replicated?](#how-does-the-information-get-replicated)
  - [Replication lag](#replication-lag)
    - [Reading your writes (read-after-write)](#reading-your-writes-read-after-write)
    - [Monotonic reads](#monotonic-reads)
    - [Consistent prefix reads](#consistent-prefix-reads)
  - [Single leader replication](#single-leader-replication)
    - [Setting up new followers](#setting-up-new-followers)
    - [What happens when a follower fails?](#what-happens-when-a-follower-fails)
    - [What happens when the leader fails?](#what-happens-when-the-leader-fails)
  - [Multi-leader replication](#multi-leader-replication)
    - [Conflict avoidance:](#conflict-avoidance)
    - [Automatic resolution:](#automatic-resolution)
    - [Multi-leader replication topologies](#multi-leader-replication-topologies)
  - [Leaderless replication](#leaderless-replication)
    - [How we write](#how-we-write)
    - [How we read](#how-we-read)
    - [Limitations of quorum consistency](#limitations-of-quorum-consistency)
    - [Monitoring](#monitoring)
    - [Sloppy quorum](#sloppy-quorum)
  - [Detecting concurrent writes](#detecting-concurrent-writes)
    - [Last write wins](#last-write-wins)
    - [Determining concurrency](#determining-concurrency)
- [Chapter 6 - Partitioning](#chapter-6---partitioning)
  - [Partitioning and replication](#partitioning-and-replication)
  - [Approaches for partitioning](#approaches-for-partitioning)
  - [Indexing and partitions](#indexing-and-partitions)
  - [Rebalancing partitions](#rebalancing-partitions)
    - [Rebalancing strategies](#rebalancing-strategies)
  - [Request routing](#request-routing)
- [Chapter 7 - Transactions](#chapter-7---transactions)
  - [Single-object vs multi-object](#single-object-vs-multi-object)
  - [Isolation levels](#isolation-levels)
  - [Concurrency problems](#concurrency-problems)
    - [Dirty reads](#dirty-reads)
    - [Dirty writes](#dirty-writes)
    - [Read skew](#read-skew)
    - [Lost updates](#lost-updates)
    - [Write skew](#write-skew)
  - [Read committed](#read-committed)
  - [Snapshot isolation](#snapshot-isolation)
  - [Serializability](#serializability)
    - [Actual serial execution](#actual-serial-execution)
    - [2PL (Two phase locking)](#2pl-two-phase-locking)
    - [Serializable Snapshot Isolation](#serializable-snapshot-isolation)
- [Chapter 8 - The trouble with distributed systems](#chapter-8---the-trouble-with-distributed-systems)
  - [Clocks](#clocks)
- [Chapter 9 - Consistency and Consensus](#chapter-9---consistency-and-consensus)
  - [Consistency guarantees](#consistency-guarantees)
  - [Linerializability](#linerializability)
  - [Ordering guarantees](#ordering-guarantees)
    - [Sequence number ordering](#sequence-number-ordering)
    - [Laport timestamps](#laport-timestamps)
    - [Total order broadcast / Atomic broadcast](#total-order-broadcast--atomic-broadcast)
    - [Linearizable storage from Total order broadcasts](#linearizable-storage-from-total-order-broadcasts)
    - [Total order broadcast from linearizable storage](#total-order-broadcast-from-linearizable-storage)
  - [Distributed Transactions and Consensus](#distributed-transactions-and-consensus)
    - [Atomic commit and 2PC](#atomic-commit-and-2pc)
    - [Distributed transactions](#distributed-transactions)
    - [Fault tolerant consensus](#fault-tolerant-consensus)


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

### 1. Through databases

- the data has to be encoded when writing (as it leaves the memory), and decoded when read
- when doing an upgrade, we can end up with the following situation:
    - old code knows about field A
    - new code knows about fields A and B
    - new code writes a record with A and B
    - old code reads it and deserializes into an object (with just field A)
    - old code writes the serialized record and overwrites the initial version, losing B

- when doing a migration, the old rows are not usually rewritten
- instead, they get a default (null) value for the newly added columns

- when doing a backup or a dump, it is good to also take the time to bring all the records to the latest schema
- since the rows will have the same schema, Avro is a good fit for compression

### 2. Through service calls

- writing to a DB allows the readers to perform arbitrary queries, so it's more flexible
- exposing an API only allows a subset of queries, i.e. those allowed by it

### REST 

- as the services are developed / deployed independently, we must support different versions talking to each other
- as opposed to a DB write / read, this is a **synchronous** communication
    - the client expects a response as soon as possible
- JSON responses usually consider extra fields added to be backwards compatible, i.e. the reader should just ignore those
- can use API versioning, in which the clients use a different URL for different supported versions
    - has the downside of forcing the server to maintain the old versions too

### RPC

- attemps to emulate a network call as if it would be local
- there's a clear disctinction between a client and a server usually, so the server can be updated first (i.e. we only need backwards compatibility)

### 3. Through message brokers

- the communication is asynchronous by design, i.e. we return once the message has been persisted to the broker
- acts as a buffer when the readers are down or too slow
- can redeliver messages to a process that has not acknowledged them (via offsets)
- removes the need for the sender to know the receiver's address
- allows fanning out, i.e. one message to multiple consumers
- decouples the sender from the receiver, abstracting the latter away (we know *someone* will consume a message, we don't care who that is)

- the communication is mostly one-way, since we don't expect a response
    - this can be emulated via another channel: another topic, or a callback etc

- no schema is usually enfored, just key-value pairs
    - any encoding format would work, including a compressed one
    - puts the compatibility responsability on the producer and consumers

# Chapter 5 - Replication

- keeping multiple copies of data on different machines, that share a network

Three main reasons:

- bring data closed to the user (to reduce latency)
- increase availability, by having redudancy in case of failures
- scale out the read capacity, through multiple machines that can serve the requests

- the difficult aspect is not in copying the data itself, but copying the changes to it, in a constent manner

Terminology:
- a replica is an instance that stores a copy of the database (or a subset of it)
- leader: a replica that accepts write requests
- follower:
    - a replica that does not accept write requests. Instead, it forwards these to the leader
    - whenever the leader performs a write, the changes must replicate to the followers
    - thus, each follower consumes the replication log of the leader

When does the replication happen:

## Synchronous replication

- the server replicates the operation to the follower(s) before returning to the caller
- guarantees follow up reads will see the most up to date information when reading from the followers
- impractical to wait for **all** the followers to acknowledge, as a single node outage would mean system outage
- a compromise exists, where we have a synchronous follower (which might be a good candidate for a backup leader), while the rest are async
    - if the sync follower becomes slow or unresponsive, another one is promoted in its place

## Async replication

- the server can return to the caller before the replication operation has finished
- the caller might get stale information when doing a follow up read from a follower that has not yet received the information
- the fastest (as it only requires one node persisting the information)
- can continue to work even if all the followers are down, as opposed to a synchronous model
- pitfall: if the leader fails, any information not propagated is lost


## How does the information get replicated?

Several options to replicate the changes between nodes:

1. Replicate the statements
    - simplest, as the leader just has to forward the operation it received
    - has to pay attention to non-deterministic results that might result in a different outcome when executed again (e.g. NOW(), RAND())

2. WAL shipping
    - the leader exports it's WAL, so the actual values that would be written to the disk, not the instructions
    - the same writes can be replayed to construct a replica, just like they would be for bringing back a single node
    - very low-level, as the WAL might reference actual pages etc. 
    - **Causes an incompatibility between versions**, so updates might require downtime

3. A **logical log** replication
    - a higher level abstraction that defines the operations against rows
    - contains enough information to replicate the operation, e.g. all information for a new row, IDs for deletion, deltas for updates etc.
    - for a transaction, we have multiple such entries, as well as delimiters for the transaction
    - decoupled from the actual implementation, so we can do rolling upgrades or use different storage engines
    - used in CDC, i.e. for exporting to a data WH

4. Let the application handle it
    - a trigger can be configured to write the changes to a new table
    - application code reads and it handles the replication itself

## Replication lag

- main issue is that replication is not instantaneous
- different nodes can get simultaneous write requests
- a node can be asked for data that hasn't yet replicated to it

### Reading your writes (read-after-write)

- usually one node is responsible for writes, while all others can serve reads

Steps:
1. Write data (lands on node A, which replicates to B)
2. Read from node C (inconsistently)

![Read after write consistency](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/read-after-write.png)

Solutions:

- read information from the node that is the writer
    - a user's writes will go to one node
    - perform the reads for that profile owner from the write replica
    - the writer will always see up to date information

- track when a piece of information is updated
    - for a while, serve the reads from the write replica
    - later on, serve from all of them (presumably giving them time to replicate the information)

### Monotonic reads

- making several read requests will show you a monotonic sequence of changes
- you won't see version 2 of the data, and then version 1 for a follow up request

![Monotonic reads](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/monotonic-reads.png)

Solutions:

- balance a user's reads to the same replica instead of randomly
- requires consistent hashing or similar to consistently send the user to the same node

### Consistent prefix reads

- if a sequence of writes happens in a certain order, anyone reading those writes will see them in the same order
- this is a problem in sharded DBs, where partitions operate independently

Solutions:

- have related writes go through the same replica
    - this way, the writes will be ordered correctly in the replication log
- vector clocks can help address this for independent writers

## Single leader replication

- one node performs the writes, all the other nodes can serve reads

### Setting up new followers

1. Take a snapshot of the leader's database at a certain offset
2. Copy the data to a new follower
3. Ask the leader for the information since the offset (log sequence number)
4. Ingest the rest of the data in order to catch up with the leader

### What happens when a follower fails?

- each follower needs to keep the current offset of the leader that it has applied to its view of the data
- when the node reboots, or the network is restored, it asks the leader for all the data since the last offset it knows about (similar to a kafka consumer)

### What happens when the leader fails?

- no simple solution
- main issues:
    - detecting the leader is down
    - chosing a new leader
    - ensuring the new leader is used by all the other nodes

Potential problems:

1. new leader might not have all the writes replicated
    - problematic with async replication
        - perhaps the sync follower can be promoted, if any exists
    - the unreplicated writes are usually discarded
    - can be a problem if those had side effects on other parts of the system, e.g. IDs that were persisted in some other storage, service etc

2. multiple nodes can believe they are the leader (**split brain**)
    - we don't want multiple nodes accepting potentially conflicting writes, as there's no mechanism to reconcile them
    - the old leader might come back and not know about the new one
    - how do we detect there is more than one leader?
        - TODO - figure out
    - how do we ensure we only demote a single one and not both?
        - TODO - figure out

## Multi-leader replication

- multiple leaders accepting writes
- each leader is a follower of the others and consumes their replication log
- a good solution for multi-DC deployments
    - each DC has one leader node and several followers
    - writes don't have to go to a single leader, which is potentially further away

- offline operations can be seen as a generic multi-leader system
    - the local client (app) can continue serving both reads and writes while offline, just like a leader
    - when going back online, it will replicate its changes to other leaders, while consuming their logs

PROS:

- the system is more robust, as losing the connection to a leader still allows writing to others
- losing an entire DC would not cause an outage, as other leaders can still work

CONS:
- need to reconcile potentially conflicting writes
- the conflict resolution has to converge 
    - all replicas should get to the same value, they can't apply the writes independently

Solultions:

- give each write a random ID and always take the one with the highest (leads to data loss)
- give each replica an ID and always take one of them's write (e.g. highest replica ID)
    - also leads to data loss
- merge the values into one (e.g. B and C become B/C)
    - would make no sense in most scenarios
- record both versions and let the application handle the resolution (e.g. merge conflicts in git)

### Conflict avoidance:

- ideally, we could avoid the conflict in the first place (what most DBs dos)
- we just need to ensure writes for one record go through the same leader
    - all of a user's posts will be processed by a one particular leader (perhaps picked on proximity)
    - we get most of the benefits, without the biggest issue (as conflicts shouldn't normally occurr)
    - problems can appear when e need to change the leader to which the writes go (e.g. if down)
        - then, the problem turns into a conflict resolution, as we could have conflicting writes

### Automatic resolution:

1. three-way-merge (like git does)
    - compares the initial version with A and B's writes
    - asks the user to resolve any conflicts

2. CRDT (conflict-free replicated data types)
    - data structures that allow for concurrent editing 
    - have built in algorithms for conflict resolution
    - uses two-way merge (compares just the conflicting versions, and not the base)

### Multi-leader replication topologies

![Multi leader topologies](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/multi-leader-topologies.png)

1. Everyone replicates to everyone
   - can result in writes arriving out of order
   - Node 1 tries to replicate a Create to node 2
   - Node 3 tries to replicate a follow up Edit to node 2
   - the Edit arrives before the Create due to delay
  
2. Each node replicates to it's left / right neighbor, in a circle
   - need to stop infinite loops: each node tags the replicated record with its ID
   - when a replication log comes in that was already tagged by the current node, stop
3. All replication goes through a central node

- For both 2 & 3 we need a workaround mechanism when a node fails (either the central one, or a link in the circle)
- The writes should arrive in order for 2 & 3, as they are replicated by the same node each time

## Leaderless replication

- all nodes can accept writes
- a client / extra role is responsible for sending reads / writes to multiple nodes
- we rely on **quorum** for consistency
- good for high-availability + low latency, at the expense of stale data sometimes

### How we write

- the coordinator sends the write request to all other nodes synchronously
- as soon as we get W confirmations, confirm the write to the caller
- this is done non-transactionally, so, if less than W nodes confirm, there will be garbage

### How we read

- the coordinator sends the read / write requests to all the relevant replicas (for that partition) in parallel. This doesn't have to be all the actual instances, but just those that should hold this partition.
    - we define write threshold - a number of nodes that must confirm the write occurred before returning success (**W**)
    - we define a read threshold - a number of nodes that must return before returning a result(**R**)
    - if the nodes return different versions, we take the latest and repair the rest

To try and avoid conflicts, we need **W + R > N**. An example for 5 nodes in total:
- we write a new version sucessfully to 3 nodes
- we attempt to read the new version from 3 nodes. 
  - For normal operations, at least one of them will have the latest information
  - We can then just pick the latest version
  - Since we know which nodes are out of date, we can also update them at this time (asynchrnously). This is called **read-repair**

- In addition to read-repair, we can have a background process that checks the data regularly
  - this is due to read-repair only working on data is read; we might have infrequently read information that can go stale for long periods
  - some of the information might never make it to a replica, thus reducing the percieved replication factor

We can fine-tune the W and R parameters:
- If we want fast writes, persist to fewer nodes and read from multiple (W = 1, R = N)
- If we want fast reads, write to more nodes and read from fewer (W = N, R = 1)

### Limitations of quorum consistency
- no guarantees of consistency if using a sloppy quorum
- concurrent writes might end up writing some of the nodes, but not all
  - mechanism for conflict resolution (e.g. LWW)
- no rollback mechanism for failed writes that succeeded on some of the nodes

### Monitoring

- For single / multi leader replication, each of them has a replication log offset.
- This can be monitored to detect scenarios where it would fall too much behind
- For leaderless, values might never replicate:
  - write to 2 nodes out of 3 and consider it successful
  - never read the value, so the 3rd one doesn't benefit from read-repair

### Sloppy quorum 

- when not enough replicas are avabilable for a write to a partition:
  - we can reject the operation
  - or have some **other** replicas accept the write
    - this other replica is not normally responsible for the current value
    - once the network interruption is resolved, the writes are fowarded to the "home" nodes = **hinted handoff**
  
- makes the writes more likely to succeed
- might cause more read inconsistencies, as we wouldn't be aware of the other nodes outside of the usual partition having more up-to-date information
  
Example: N = 5, W = R = 3.

1. Try to write to 1/2/3, but 3 is down and 4/5 are slow
- write the value to 1/2
- write the value to node 6 instead of 3
2. 3 comes back online, 4/5 stop being slow
3. Try to read from 1-5
- get stale data from 3/4/5, as we don't know about 6 (not usually working for this partition)

## Detecting concurrent writes

For both multi-leader and leaderless replcation, we need to reconcile "concurrent" writes.
- An essential goal of the DB is to offer consistency, even if eventually
- This means that the values need to converge to the same on all the replicas
- A situation where one replica would authoritatively say A while another would say B should be avoided

### Last write wins

- relies on providing "some" order to the two operation
- assign each write a timestamp, and always pick the latest one
  - this won't necessarily consider them in the order they were sent, but it allows all replicas to pick the same value
  - will lead to a **silent data loss**: we accepted both writes and returned success, but now one value is overwritten (think WATM profiles during replication)
  - using UUIDs as the keys is enough to prevent conflicting writes

### Determining concurrency

- two requests don't have to actually be concurrent in time for them to cause a replication problem
- they just have to happen before replication gets the chance to be executed

Three ways to order two events:
- A definitely happened before B
- B definitely happened before A
- they were concurrent (neither knows about the other)

An algorithm that can tell the options apart allows us to use LWW for a clear ordering, while leaving the concurrent conflict resolution to the application.

TODO - add more info here: Version vector, vector clocks?
https://queue.acm.org/detail.cfm?id=2917756

# Chapter 6 - Partitioning

What?
- splitting data such that each record belongs to a single partition

Why? For Scalability: 
- different partitions can live on different nodes in a shared-nothing clusted
- scales the query load across multiple instances
- single instance queries can run in independently
- queries across partitions can be executed in parallel

## Partitioning and replication

- they are usually orthogonal
- work together to ensure scalability and reliability

- the data for a partition should have multiple copies, thus we need to replicate it
- to avoid write conflicts, we can have each node be the leader for some partitions
  - it is also a follower for the others assigned to it

![Partitioning and replication](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/partition-replication.png)

## Approaches for partitioning

- the goal of partitioning is to spread the query load **evenly**
- poor partitioning can lead to **skew**, that is one partition taking much more than others => **hot spot**

We want to partition a large set of key-value pairs. Thus, we need to let each partition handle a range of the keys.

To help with rebalancing:
- we split the range of keys into much more than the nodes (similarly to virtual nodes in consistent hashing)
- each node takes a set of partition
  - when rebalancing, the keys stay in the same partition
  - the partition get moved around as a unit

There are two main approaches:

1. Using the key directly

- the ranges are not evenly spaced, as there might be more load towards one of them (unless using random keys)

PRO:
- range queries are easy, since adjacent values should live in the same partition (with some edge cases)
  - we get this by keeping the information ordered (like a sort key in Dynamo, i.e. an SSTable)

Cons:
- need some manual intervention to break down the intervals based on the skewing
- certain access patterns can lead to hotspots
  - e.g. writing data using the timestamp as a partition key will result in a single instance taking all the writes


2. Using a hashing of the key

PROS:
- introduces randomness into the partition assignment, so it should alleviate write hotspots
  - this works as long we don't have too many requests coming in for the same key, i.e. Trump twitter
  
CONS:
- we lose the ability to do range queries (at least over the partition key)
  - since consecutive values should end up on different replicas, we'd have to query all of them
  - a compromise is to allow for range queries *within* a partition, e.g. via sort keys in Dynamo (or Cassandra)

Hotspot solution - appending some random element to the key (within a range)
- distributes the queries to multiple partitions, as they would have different hashes
- downside is we need to do multiple queries when reading (for every known preffix/suffix)
- we only want to do this for a small set of keys, so we need to keep track which those are

## Indexing and partitions

Once we split the information into partitions, we can look at how the indexes are stored.
Two approaches emerge:

1. Local secondary index (document based partitioning)

- maintains one secondary index stucture for each partition
- the local secondary index contains information about the elements **in its partition only**. 
- this helps keep the partition strongly consistent (e.g. local indexes support consistent reads in dynamoDB)
- to query the secondar index, we need to query **all** partitions
  - call *scatter-gather*, this is a very expensive operation, as we need to wait for all of them to return (long tail latency)

![Local index](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/local-index.png)

2. Global secondary index (term based partitioning)

- an alternative is to store all the information about e term (e.g. red cars) in a single partition. 
- the index might be too large to live in a single partition, so different *terms* need to live in different partitions
- we can partition the index by its keys (good for range queries), or by hashes of the keys(better balancing between partition) 

PROS:
- this makes reads more efficient, as we only need to look in a single place

CONS:
- writes are more complicated:
  - a document might have attributes that are part of several indexes
  - their partitions might be different, so the write needs to touch multiple nodes
  - thus, it is usually eventually consistent (like dynamoDBs)

![Global index](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/global-index.png)

## Rebalancing partitions

- inevitable process, so it must be accounted for
- goals:
  - evenly distributed load between partitions
  - availability during the rebalancing
  - minimize amount of values that need to be moved between nodes

### Rebalancing strategies

0. Mod N

- bad, increasing or decreasing N causes most of the keys to move (e.g. mod 10 vs mod 11)

1. Fixed number of partitions
   
- a predefined number of partitions, chosen at the start
- should be much higher than the number of nodes
  - thus, each node holds multiple partitions
  - when a new node joins, it takes entire partitions from the older ones
  - limited to having max_nodes <= partition_count

2. Dynamic partition by data size

- keeps the ratio of data_size / partition_count relatively constant
- the DB engine decides (very similar to B-trees):
  - when a partition is too large, it gets split into two halves (and moved to another node)
  - when two partitions are too small, we merge them into a single one
- initial setup needs an additional help with partitioning, so it doesn't start with a single one
- MongoDB supports both key and hash dynamic partitioning

3. Dynamic partition by node count

- keeps the ratio of partition_count / node_count relatively constant
- when a node joins:
  - it picks some random partitions from the other replicas
  - takes half of each of them

## Request routing

- once the data is split into partitions, we need a way to determine where to direct the read / writes for a particular key
- the question boils down to: who knows which partition is assigned to which node?
  
Three main approaches:

1. Any node accepts requests and fowards them
- in effect, the nodes themselves keep this mapping

2. A routing tier accepts the requests
- this is responsible for fowarding the request to the right node

3. The client sends directly to the right node
- client needs the knowledge of which partition goes where

![Partition assignment](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/partition-assignment.png)

Zookeeper is a good candidate to store the authoritative mapping between key ranges and partitions / nodes.
- each node registers itself with zookeeper
- each consumer registers for notifications, and gets called when the partitions change

# Chapter 7 - Transactions

## Single-object vs multi-object 

- most DBs offer isolation at the row level
  - this means we either update an entire value / document, or nothing
  - we also won't see a partially written document
  - this can be done by write-and-swap: 
  - instead of updating a B-tree node, we write another copy with the new value / pointers
  - the new tree is then swapped atomically

- some scenarios when multi-object is useful:
  - needing to keep multiple copies in sync: esp useful when the data is denormalized
  - keeping FKs up-to-date

- we shouldn't always rely on retrying the operation all the time:
  - a mutating retry might have side effects
  - the server might fail to respond due to load, which we'd be adding to

## Isolation levels

Transaction isolation attempts to abstract concurrency: we can pretend no concurrency is happening, and the effects should be as-if the operations ran serially.

A serializable approach offers the strongest guarantees, but that comes at a performance cost. Most DBs don't offer it by default.

Various weaker isolation levels exist. Each offers more guarantees than the previous, but at a peformance hit cost.

## Concurrency problems

### Dirty reads

- concurrent transactions can see each other's non-commited writes
  
![Dirty reads](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/dirty-reads.png)

| Isolation level    | Protects against |
| ------------------ | ---------------- |
| Read uncomitted  | No    |
| Read committed | **Yes**     |
| Snapshot isolation    | **Yes**    |
| Serializable    | **Yes**    |

### Dirty writes

- concurrent operations can overwrite each other's non-commited data

![Dirty writes](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/dirty-writes.png)

| Isolation level    | Protects against |
| ------------------ | ---------------- |
| Read uncomitted  | No    |
| Read committed | **Yes**     |
| Snapshot isolation    | **Yes**    |
| Serializable    | **Yes**    |

### Read skew

- transaction A might read row X and get 100
- transaction B comes and changes row X to 200
- transaction A reads row X and now gets 200

- multiple rows might be involved, which can lead to an inconsistent view of the table (i.e. it was never in that actual state)

- read commited is not sufficient, as we're always reading commited transactions
- the data however changes during the transaction

- also called *non-repeatable* read, as a follow up attempt would get the right value
- particularly problematic when 
  - doing backups (as we'd get half of the old state, half of the new)
  - running integrity checks

![Read skew](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/read-skew.png)

| Isolation level    | Protects against |
| ------------------ | ---------------- |
| Read uncomitted  | No    |
| Read committed | No     |
| Snapshot isolation    | **Yes**    |
| Serializable    | **Yes**    |

### Lost updates

- occurs when we need to read the value before modifying it
  
Examples:
- incrementing a counter
- adding a value to a list in a document
- text modifications

![Lost update](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/lost-update.png)

| Isolation level    | Protects against |
| ------------------ | ---------------- |
| Read uncomitted  | No    |
| Read committed | No     |
| Snapshot isolation    | No    |
| Serializable    | **Yes**    |


1. Some DBs support atomic operations:
- increment: SELECT counters SET value = value + 1 where key = 'foo'
- MongoDB provides atomic operations for modifying a part of the document
- Redis has atomic operations for some data structures
- alternatively, serialize the operations on a single thread, to guarantee their ordering

2. Explicit locking can also be used:
- the application layer must explicitly tell the DB to take a lock on a subset of rows
- this is done via the SELECT..FOR UPDATE command, which gets a lock on all returned rows

3. **Compare-and-set** is also an option:
- read the value
- send a conditional update based on the read value (or a version)
- if the condition fails, the application layer knows the data has changed and can retry the operation

### Write skew 

- two concurrent transactions read and update the same objects
- each of them makes a decision based on the snapshot, which might not be valid anymore
- a generalization of the Lost-Updates problem
- usually DB support is helpful via constraints

Solutions:
- lock the rows where the condition is applied
- in some cases, nothing to be locked (i.e. select returns nothing)
  - introduce an artificial lock => **materialing the conflicts**
  - split down the constraint space into a set of locks
  - use these locks to force serializability for the competing operations

Example: 
1. booking a resource
- we check if a resource is booked already
- if not, we acquire it
- since this is a two-step process, the initially read information might be outdated by the time we try to insert

2. Multiplayer game:
- users simultaneously moving an item into the same location

3. Unique username / DNS name etc
- users simultaneously creating a resource that has to have a unique name

![Write skew](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/write-skew.png)

| Isolation level    | Protects against |
| ------------------ | ---------------- |
| Read uncomitted  | No    |
| Read committed | No     |
| Snapshot isolation    | No    |
| Serializable    | **Yes**    |

## Read committed

- weakest isolation level, but the most common
- prevents dirty reads/writes

- writes are prevented by acquiring a lock on the row
- since a single transaction can be modifying a row at one point, we can store the initial value, as well as the in-progress modified one
  - any transactions reading the value will get the old one
  - once the writing transaction commits, we swap the two values in a single operation

## Snapshot isolation

- designed to address Read-skew
- the idea is for the tranactions to operate on a consistent view of the database

- writes are implemented by acquiring a lock on the row (so they block each other)
- generalizes the concept of storing multiple versions for a single row (called MVCC - **multi-version currency control**)
  - multiple in-flight transactions might need to see different versionsn of a row
  - when a transaction asks for a value, we need to determine which of the values to return

Essential steps:
- transactions get a monotonously increasing Id
- when a transaction changes the data, we tag the information with its Id

Algorihtm:
- when a transaction starts, we determine which previous transactions have already finished (e.g. 1,2,4,5)
  - this list doesn't change during the execution, e.g. 3 finishing a bit later will be ignored
- when reading a value, we walk back through the verision history until we find one that was written by a transaction in our list
- follow up transactions that commit will be ignored (as our list is immutable)

Alternative:
- by using a copy-on-write mechanism for the B-tree, each transaction can get its immutable copy of the tree
- no need to tag the information, as subsesquent transactions won't alter an older one's copy of its tree

- solves the read-skew problem, but write-skews can still happen (especially with conditions spanning multiple rows)

## Serializability

- forcing the trsansactions to run in an actual sequence offers the strongest isolation guarantees (as there's no concurrency)
- three main ways of achieving this:
  - executing the operations serially, on a single thread
  - Two-phase commit
  - Serializable Snapshot Isolation

### Actual serial execution

- some subset of data systems support it: Redis, VoltDB etc
- the throughput is limited to that of a single CPU core
  - to actually scale the system, we rely on partitioning the data across multiple shared-nothing nodes
  - each node can continue to execute trsansactions serially, as they would only operate on local data
  - sometimes this is not possible, especially with multiple global secondary indexes
- this is fesible when the dataset is in memory - reading from the disk in a single-threaded app would kill its performance


- over HTTP, a transaction does not span multiple requests
  - e.g. in Dynamo, we gather the different puts and send a single HTTP request that represents the transaction
  - this helps avoid long-lived transactions
  - helps avoid multiple trips to the DB, with an un-bound delay between them
  - in effect it becomes a stored procedure: either imperatively (Redis and Lua), or declarative (dynamo)
  - if we enfore the transactions being deterministic, replaying them will result in the same state
    - thus, we can use this replay mechanism for replication

![Http transaction](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/http_transaction.png)

### 2PL (Two phase locking)

- forces the actual serial execution when it detects two transactions might intefere with each other (by reading or writing the same row)
- first phase: acquire the locks
- second phase: execute and release them

- if transaction A has read an object and transaction B wants to modify 
- it, B must wait until A has finished (reads block writes)
  - ensures B can't change the data while A has a view of it
- if transaction A has written to an object and B wants to read it, B must wait until A has finished (writes block reads)
  - ensures B will use the latest data when it actually executes

Algorithm:
- each object in the DB has an associated lock
- locks have two modes: exclusive or shared

- when performing a read, an operation tries to acquire a shared lock
  - this works, unless another transaction has an **exclusive** one
- when performing a write, an operation tries to acquire an exclusive lock
  - the transaction must wait if any other operation has **any** type of lock on it

Drawbacks:

- **Can lead to deadlocks**
- a single transaction (or one with many locks) can slow down all the others

Predicate locks:
- to prevent phantoms, we must sometimes lock on rows that don't yet exist (e.g. in a booking system)
- another approach is to create a **predicate lock** - a lock that applies to any rows returned by a query

### Serializable Snapshot Isolation

- an optimistic concurrency mechanism
- does not lock anythign and allow potentially competing transactions to continue
- if a conflict is detected, the latter transaction is aborted
  - if this happens often, the performance can suffer (as it's peforming work just to roll it back)

- relies on snapshot isolation, with an added mechanism to detect stale assumptions:
- if a transaction reads data that was modified between read time - commit time, we assume it to be stale
- two scenarios where this can happen:

1. a write transaction was in progress when the snapshot was taken (but not yet commited); if this gets commited, the read is stale

  ![MVCC stale read](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/ssi-write-mvcc.png)

- Since we keep track of the different versions, we know transaction 43 might have acted on stale data, as it read a version of the table < 42. 
- we just need to keep track of which transactions were ignored by the reads
- When 43 wants to commit, we compare the list of ignored writes to the list of transations that completed already

2. a write transaction came in after the snapshot and commited a different value

  ![Write after read](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/write-after-read.png)

- we can't use the previous mechanism, as we'd have to look back in time
- instead, a similar mechanism to the locks in 2PC can be used
  - when a read consumes some values, mark that with a read lock
  - if a write comes in afterwards, it can know which other transactions are consuming the data it's changing

# Chapter 8 - The trouble with distributed systems

- since we're relying on a set of machines connected by a network, communication is never instantaneous.
- delays can and will happen for various reasons outside of our control
- it's up to the application to be able to hanlde this => we're building a reliable system from unreliable components

- no accurate way to determine appropriate values for timeouts
    - we should experiment until we find the right ones for the situation

## Clocks

Important: DO NOT TRY TO CORRELATE TIMESTAMPS ACROSS MACHINES

- cannot rely on them having a similar state across multiple machines
    - thus, ordering by timestamps taken across multiple machines is meaningless
- NTP (network time protocol) can help, but not too much

1. Time of day

- measure time like a human would
- usually synchronized with NTP
    - this can mean it might get corrected if too far behind / ahead => time jumps
    - cannot be used to accurately measure duration, as it might be negative

2. Monotonic clocks

- guaranteed to always move forward
- has no further meaning associated with it, so do not compare values between machines
- good choice for measuring timeouts (on a single machine)

- monotonic IDs are particularly relevant for Snapshot Isolation, where we need to know whether a transaction should be visible to others
    - easy on a single machine, challenging in a distributed system

# Chapter 9 - Consistency and Consensus

## Consistency guarantees

Similarly to transactions, there's a tradeoff between the strength of the consistency guarantees vs performance.

## Linerializability

- called strong consistency
- strongest level of consistency
- makes the system appear as if:
  - holding a single copy of the data
  - all operations are atomic against it
  
- this allows the application to abstract the idea of multiple replicas

- once a client completes a write, all follow up reads must return the new value
- "concurrent" operations can be undefined

- once a read returns a new value, any follow up operations started **after** the first one finished need to also return the new value
  
![Linearizable system](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/linearizability.jpg)

Linearizable systems:
- actual serial execution => actions are linear by design
- 2PL => locks imply we get the right results

Non-linearizable:
- SSI, as it relies on Snapshot isolation, so it will see out of date information

Systems that require linearizability:

- leader election
  - all nodes need to agree on a single value => zookeeper

- acquiring distributed locks 
  - a single one must be available at all times

- constraints and uniqueness

## Ordering guarantees

- we rely on ordering to determine causality: A happened before B, so they might be correlated
- if two operations are happening "concurrently", they should have no relation to each other

Causality examples:
- consistent preffix reads
- multi-leader replcation: an update notification arriving before a create notification

**Causally consistent** - seeing one piece of data implies seeing all the data that preceeded it. (but not necessarily all data that came before it chronologically)

**Total order** - any two events can be ordered chronologically, e.g. kafka events in the same partition

Consistency models:

1. Linearizability = total ordering = system behaves as is there's a single copy of the data + operations are atomic
- there are no concurrent operations in a linearizable system, as everything has to be on a single timeline

2. Causality = partial ordering = some events are ordered (if they are causal to each other), while others are incomparable 
- in practice, this is the most widespread model
- does not sacrifice performance and availability, especially for distributed nodes
- we need to provide the following:
  - concurrent operations can be processed in any order
  - when a replica processes an operation, it must make sure it also processed all the operations that happened before it
  - version vectors can be used to acomplish this

### Sequence number ordering

- the goal is to assign "some" value to each operation, such that causal operations have incrementing values
- we cannot use time-of-day timestamps due to jumps, but we can use a monotonous sequence that gets incremented on each operation
- these are compact and provide a total ordering, i.e. any two can be compared
- we can assign these such that they also imply causality:
  - if A happened before B, A's id will be smaller than B's
  - if A and B happened concurrently, no guarantees, but the 2 operations cannot be correlated

- When there's a single leader, it is sufficient to apply the replication log in the order in which it was emitted.

- When there are multiple generators (e.g. due to multiple leaders or partitioning), various approaches that boil down to spliting the ranges:
  - each generator has its own disjunct set of Ids
  - we get an ordering guarantee within that set, but not across them
  - simiarly to Kafka and the ordering guarantees within a single partition, but not across them

### Laport timestamps

- each node has an ID and a counter value
- when comparing 2 operations: 
  - the one with the highest ID always comes later
  - in case of ties, the highest nodeId comes later

- on each operation, the server increments its internal counter
- the internal counter is returned to the caller
- the callers include the latest counter in the follow up requests - this is by design the highest they have seen for that value
- when a server receives a counter higher than its own, the internal one gets overwritten

While this offers us the ability to compare any 2 operations, it does so only once they have finished.

This is not sufficient when trying to decide on resource allocation, e.g. a unique username. 
- for that, we'd need to know if our counter is up-to-date
- that would imply querying all the other nodes, which is not feasible

![Lamport timestamps](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/lamport-timestamps.png)

### Total order broadcast / Atomic broadcast

A protocol that fulfills the following:
- the distributes all messages, in the same order, to all the nodes
- has no relation to causality, as we don't place any constraints on the actual ordering of the messages

- this is a very useful property for DB replication
- every replica is guaranteed to process the replicated operations in the same order

- while messages cannot be deliver while a replica is down, they will be delivered in the correct order when it comes back up

Applications:
1. DB replication 
- if all nodes agree on an order of operations => all operations are applied in the same order

2. Serializability
- each transaction is deterministic
- all nodes process them in the same order => ????

- we can look at total order broadcast like an append only log
  - when a message is delivered, it is appende to the log
  - all nodes must deliver the messages in the same order

- can be used as a fencing token generator:
  - each request to acquire a lock is appended
  - it gets a monotonously incrementing value
  - this value can be used as the fencing token

Zookeeper provides an implementation:

- there is a FIFO connection between any two instances
- uses TCP for the ordering guarantees:
  - if a packet is delivered, all the ones before it have also been delivered
  - once a channel is closed, no other messages can come through

### Linearizable storage from Total order broadcasts

Trying to register a unique name - Assuming we have a total order broadcast protocol:

1. Write a message tentatively reserving the username to the log. Note: this does not guarantee you actually get it!
2. Read the log until you encounter your message
3. Check if any other message in between claimed the username
   - if none, you actually commit and take the username
   - otherwise, abort

Why this works:
- log messages are delivered to all nodes in the same order
- all nodes will share the same view and have a consisten view of the ordering
- for several concurrent writes, we can easily tell which one was first

- this guarantees writes are linerizable
- we can read from a replica that is somewhat behind, and not see the latest data

To offer linearizable reads:
0. Read from a replica that has synchronous writes
1. Wait for the current replica to be up to date:
   1. Add a message to the log and wait for it to be received
   2. OR Query the leader for the offset and wait until it is delivered
   - At this point, we have the latest data for that point in time, so we can accurately respond

### Total order broadcast from linearizable storage

Assume we have a linearizable storage:

1. We use a counter that can be incremented atomically
2. When an operation comes in, we mark it with the current value and increment it
3. The message is sent to all other nodes, tagged with the ounter value
4. The message sequence is a continuous range of values
   1. If a node receives message 5 but misses message 4, it has to wait until it gets the missing one
   2. Missing messages have to be resent, but the ordering is persisted

The problem is reduced to having a counter value that is consistent across a distributed system
=> **Total Order Broadcast = Linearizable Storage = Consensus**

## Distributed Transactions and Consensus

- consensus is required for leader election
- distributed transactions also need some form of it, e.g. we need them to succeed on all the nodes (or abort)

### Atomic commit

Flow for a single node:
- data is written to the disk
- the commit message is also written -> this is done atomically, and represents the single point of no return
  - if the commit message is successfully written, the data WILL be persisted, even across a DB crash
  - if anything happens before the message is written, the transaction is in effect discarded

When dealing with 2 (or more nodes):
- we can't pick the moment in time when the first one commits as the point of no return
  - this is due to the fact that we don't know whether the 2nd one will succeed

- multiple reasons why one node might commit and another abort
- if that's the case, we end up with a permanent inconsistence between the nodes
- once a transaction is committed (on any node), other transactions will start seeing it and might rely on that data
  - thus, we cannot cancel a transaction once it is committed

=> **a node must commit only if it's sure ALL other nodes will commit too**

### Two-phase commit

- process for providing atomic commits within a distributed database
- requires an extra role of **coordinator** -> can be part of the library

![2PC](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/2PC.png)

Steps:

1. client sends data to the nodes
2. client initiates a commit 
- the coordinator asks all nodes to **prepare**
- each node either responds:
  - with a failure
  - with an approval => means the node promises to commit the transaction if asked and cannot refuse / fail
3. if all nodes approved, the coordinator writes the commit message to its log -> commit point
4. the commit request is sent to all participants
  - if the request fails or timesout, the coordinator must retry forever
  - a node failing implies it has to commit the transaction before it recovers

If the coordinator fails:
- before sending out the *prepare* message -> each node can abort their transaction
- after sending out the prepare -> a node that responded with a promise during the prepare cannot unilaterally abandon it, or commit it (as it doesn't know the future)
- the coordinator coming back has to interpret the log and retry

![2PC failure](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/2pc-coordinator-failure.png)

### Distributed transactions

1. Single system transactions:
- distributed transactions within the same DB, just spanning different nodes
- often feasible, see dynamoDB (with some limitations)
- the technology controls the protocol and it's shared by all the nodes

2. Multi-system transactions:
- participants from 2 or more technologies
- much more difficult to provide atomicity
- requires that all participants support the same atomic commit protocol
  - all nodes participate in the same 2PC protocol
  - if one system does not support it, we will end up with an operation executed multiple times

X/A transactions are one standard that seems to be supported, at least for relational DBs
- offers the building blocks / API calls for 2PC acorss multiple systems that support it
- same problem as 2PC, as a participant can be stuck forever with rows locked
- requires manual intervention to commit or rollback
- does not scale, not very interesting in this context

### Fault tolerant consensus

- we saw with 2PC that the coordinator is a single point of failure
- there can be cases when nodes are stuck indefinitely waiting for the coordinator to come back

A **consensus** algorithm must fullfil the following:

- all nodes agree on the result
- each node decides a single time
- if a node decides value V, it was proposed by some node (no hacks)
- the algorithm must be finite, assuming a majority of nodes are working
  - if there's only a minority of nodes online, the algorithm is halted
  - they will not reach a consensus (instead of reaching a wrong one)
  - this is due to the fact that the other partition might have majority and might decide something else

- Most algorithms descide on a **sequence** of values that satisifies the properties above. 
- This turns it into a total order broadcast solution:

1. each round, nodes proposes a value they want to send next
2. they all agree to which value should be the one broadcast

- total order brodcast = multiple rounds of consensus, with each round sending out a single value
- all nodes decide to deliver the same value
- messages are not duplicated / each node decides just once

### Epoch numbering

- we provide a weaker guarantee compared to "a single leader" => "a single leader per election cycle / epoch"
- epoch numbers are totally ordered and monotonically increasing => we can always pick the leader from a later election to be the true one => no more split brain
- to be elected, each leader needs a vote from an absolute majority of nodes (e.g. 3 out of 5 in total)

Compared to 2PC:
- leaders are not manually selected (unlike a coordinator)
- no need for **all** nodes to agree to commit, a majority is sufficient
- we have a recovery mechanism in place for leader failure

Drawbacks:
- leader election takes time from actually serving the requests
- might end up in a scenario where it's all we do

### Raft discussion

## Membership and coordination

- hold small amounts of data, operating on it in memory
- persisted to the disk for durability
- the data is replicated across all nodes, using a fault-tolerant total order broadcast
  - this means we have a consensus across all the nodes

Other supported operations:

1. Linearizable atomic operations => distributed lock
  - the lock is actually a lease, since it has an expiry time
2. Total ordering of operations
  - when a lock is acquired, the operation gets a fencing token (monotonously increasing operationId: zxid + cversion)
  - this can be used to prevent issues arrising from process pauses
3. Failure detection
  - client and server exchange heartbeats
  - when no heartbeats received for a predetermined period => node is dead => all locks held by it are released
4. Change notification
  - clients can subscribe to events related to changes on a node
  - since other clients joining / leaving result in a node change, this can be a notification service for changes in membership

  Examples of uses:
  - when a node joins / leaves, we might need to do partition rebalancing
  - knowing when this happens without having to poll is efficient
  - service discovery: zookeeper can be used instead of DNS if we require consensus / more up-to-date information (no caches)

  Note: we can use Zookeeper for automatic failover detection / correction.

# Derived data

We distinguish each piece of data / system storing data in one category:

- source of record
  - holds the authoritative copy of the information
  - when new data comes in, it is written here
  - each fact is represented just once

- a derived data view
  - takes data from a system of record and transforms / processes it
  - can always be recreated from the original source (e.e. a cache)
  - is used to offer diferent views of the same data, e.g. an index

**Always a good idea to clearly differentiate which data store holds which type of data for our system.**

# Chapter 10 - Batch processing

Different types of systems, based on their response times:

1. Real time (user facing)

- the user expects a response
- when the service receives a request, it tries to fulfill it as soon as possible
- *performance measure: response time and availability*

2. Offline systems - batch processing

- takes a large amount of data as its input
- after some time, produces its output
- jobs can be long running, without anyone waiting for their output
- triggered by schedulers
- *performance measure: throughput (how long the jobs takes based on some input)*

3. Near-real time (stream processing)

- consumes inputs and produces outputs (similarly to a batch system)
- is not triggered / awaited by a user directly 
- is run based on events coming in, shortly after they happen


# Chapter 11 - Stream processing

- as opposed to batch processing, it allows us to handle an unbound amount of data, without splitting it into artificial chunks.
- batch processing must define precise intervals at which to run, so it inevitably introduces a delay.
- stream processing is simply batch processing with a very small window size (length goes towards 0)

Some properties of events:

- an **event or record** is an small, self contained and immutable piece of information. 
- it usually contains a timestamp of when it happened
- it can be encoded in various ways
- it is written once by a producer
- can be consumed by multiple consumers (the producer doesn't have to be aware who those are)

## Message brokers

- a dedicated "database", optimised for handling message streams
- allows producers to come and go (e.g. server-oriented vs peer-to-peer)
- the problem of durability is offsourced to the broker, once it accepts a message
- they allow buffering the events, in case they can't be consumer fast enough
- by definition they represent an **async** process - the producers do not wait for the consumers to process a message


Differences compared to a DB:

- designed to not stored data indefinitely 
- if data is deleted as soon as consumed, it should be safe to assume the set is relatively small compared to a DB
- instead of indexes, other mechanisms allow consuming just a subset of data (e.g. topics, partitions)

## Patterns of delivery

Two scenarios are supported:
1. dividing the load 
  - each message is delivered to a single consumer that is part of the group
  - this allows the a consumer (group) to scale horizontally
  - we can add more consumers to the group if the processing is slow

2. fanout
  - each message is delivered once to each consumer group
  - the consumer groups represent a separate conceptual consumer
  - e.g. both system A and system B want to be notified when an event comes in
  - allows multiple unrelated consumers to subscribe to the same set of notifications

These can be combined (e.g. in Kafka):
- a consumer group represents a logical "consumer", e.g. a service
  - messages are delivered **once to each consumer group**
- within that group, we might have one or more actual consumers  
  - **only one of the actual consumers receives the message from the broker**

  ## Acknowledgements and redelivery

  - message is considered to be processed only once it has been acknowledged
  - if the processing takes too long / times out, the message is delivered to another consumer
    - in effect, a message **might** be delievered multiple times