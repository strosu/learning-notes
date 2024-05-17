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


