# Designing Data Intensive Applications

My notes on Martin Kleppmann's book, with some personal additions on the topics he explores.
The notes are by no mean exhausitve and skim over areas that I find obvious based on my experience, so YMMV. 
Images are taken from the book and from online talks by Martin Kleppmann. 

Contents
========

 * [Storage and retrieval](#storage-and-retrieval)
 * [Replication](#replication)
 * [Partitioning](#partitioning)
 * [Transactions](#transactions)
 * [Problems with distributed systems](#problems-with-distributed-systems)
 * [Distributed consensus](#distributed-consensus)
 * [Stream processing](#stream-processing)

Particular implementations
========

 * [Distributed Task Queue](#distributed-task-queue)


# Storage and retrieval
---

A database needs to do two things: persists some value when you write to it, and return that value later when queried.
Two main approaches:
- log based storage
- page oriented storage

## Log based storage
 
- uses a **log**, which is an append only file
- all the writes happen at the end of a file, so they are very cheap, compared to random disk seeks
- results in an append only sequence of records
- writes is O(1), however reads are O(N), where N is the number of records
- to optimize for reads as well, we need an additional structure, known as an **index**. This stores some additional metadata, which helps us locate the data we want faster
- maintaining addtional structures impacts the performance of writes, as they need to be updated when the data changes
- databases usually don't index everything by default, but instead let the developer chose indexes manually

### Hash Indexes

- can be used for key-value storage, similarly to a dictionary
- we append the keys and values to the log
- the in memory dictionary also stores the offset of the key-value pair in the log file
- writes are still O(1), while reads also become O(1)
- can come in handy for a system that receives many writes, but over a relatively small set of keys. For example, storing view counters for videos, etc, where the number of events is greater by an order of magnitude or more compared to the number of keys (the videos themselves)

Continuously appending to the file will eventually result in the disk getting full. To prevent this, we need to compact the logs:

- when a segment reaches a certain size, mark it as closed and start a new one
- previously closed segments are **compacted**:
    - segments themselves are no modified, so we create a new file
    - this file only stores the most recent change for a particular key; in our case, the change itself reflects the entire entity (no partials)
    - thus, it is safe to only keep the most recent value and older ones can be discarded
    - once the merger is complete, reads can address the resulting object and old ones are simply deleted
    - this is done by a background thread, so reads and writes can proceed *normally*. However, as they are sharing the same resources (mostly disk bandwidth), eventually the background operations might slow down customer facing requests
- when reading, we check the most recent's file hash index
    - if the value is present, we simply use the offset to return it
    - otherwise, move to the next older hash index

Deleting records - since we can't modify previous values, a special record named a *tombstone* is used. This is then interpreted during the compaction as an instruction to discard all values for that key.

Crash recovery:
- by being in memory, the hash index can be lost
- it can be restored by reading the entire log, but it's an expensive process
- instead, flush it to disk regularly and load from there

Downsides of hash indexes:
- the need for additional space to store the index. This can become a problem if the hash table is too large to store in memory
- range queries are not very efficient, as you can't perfom efficient range queries against a dictionary

### SSTables and LSM-trees

![SSTable](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/sstable.png)

Instead of storing the records in the order in which they arrive, we can store them sorted by the key. 

Advantages:
- log compaction is much simpler:
    - very similar to merge sort
    - we have several sorted sets that need to be merged
    - at each step, compare their heads
    - if it's a duplicate, just keep the latest value (from the latter segment)
    - otherise, add the value to the compacted log
- instead of keeping all the offsets in memory, we can keep a very sparse set
    - if the key we're looking for is in the set, we're done
    - otherwise, find it's smallest/largest neighbor, which gives you a start and an end in which the value can reside
    - scan the range (either linearly, or vinary search if feasible etc)
    - each of the entries in the in memory hashset can point to the start on a segment; thus, the search will tell us the right segment containing our value (if any)

Thus, an SSTable is a disk structure, consisting of:
- the data itself (ordered by key)
- an index from keys to their offsets in the file
- a bloom filter of the data in the SSTable. This allows us to quickly (and with very little overhead) to check if a value is present in the SSTable

How an LSM tree works:

- when a writes comes in, it's stored in an append only log. This is done to protect against crashes, when the in-memory data would be lost
    - it is only used for restoring the node and is compacted / garbage collected regularly
- after it is persisted, it is added to an in-memory selef balancing tree (RedBlack tree etc) called a **memtable**
    - this allows inserting in any order and retrieving the sorted set later on
- when the memtable gets too big, it is flushed to disk and it creates a new SSTable   
    - this file contains the sorted keys from the memtable, as wel as additional information we might want to write
    - as we're writing to the disk sequentially, it can support a very large throughput
- when a read arrives, try to serve it from the memtable
    - if the key is present, we know it's the most recent value (as the time ordering from most recent to oldest is Memtable -> SSTable 1 -> SSTable 2)
    - if the key is not present, look at the most recent SSTable, then the next one etc
- from time to time merge older SSTables (fairly easy, see above advantages)

## B-Trees

![SSTable](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/btree.png)

- represent a type of page-oriented storage
- keys are also stored in order, but the sets are broken down into fixed size blocks called *pages*
- each page can be identified by its location, which allows other nodes to reference it, creating a tree of pages
- each node can contain multiple values, in effect multiple ranges. For example. if the root stores 1, 3, 5, 10 and their addresses, the next layer will be a node containing values between 1-3, another for 3-5 and so on. This can be nested and it allows for searches similar to binary search
- each node is responsible for a continous range of keys, but it might not be full. If a node can contain the values between 100-200, it might contain just some of them
- the keys stored in the node itself act as boundaries

- When reading, we traverse the tree in order to find the leaf storing the current value. 
- When updaing a key, the approach for finding the key is similar to reading. Once the leaf is found, we change the value in the page and write it back to the disk. 
- Adding a new key, we find the relevant page and append to it. If the leaf node does not have enough space, we split it into two and update its parent. This operation might need to bubble up, in which case we might end up needing to add an additional level (although that should happen rarely).

As opposed to log based systems, we might need to modify several pieces of data as part of the same operation. The simplest example is splitting a page as above. The entire operation needs to be atomic, so lightweight locks are used for this. Additionally, in order not to end up in an inconsistent state, a **write-ahead-log** is also used. This is an apped only log file, to which every incoming operation needs to be added before being applied to the pages themselves. 

## LSM vs B-tree comparision

### LSM advantages 

- LSM trees generally are faster to write to (as we're only appending), but slower to read from (as we need to query several SSTables)
- In each of the systems, a single write command results in multiple actual disk writes:
    - for B-trees, we write at least twice: once in the WAL and once in the tree itself, with an extra overhead for splitting pages
    - for LSM trees, we write just once during the operation; however, the information itself is sometimes rewritten during log compaction. Not all entities will be rewritten, as some of them might be dropped due to newer data. The only way to determine what is best is to benchmark.
- LSM trees are more compact, as B-trees might have unused portions of pages

### B-trees advantages

- for LSM trees, sometimes the background process of compaction can interfere with the incoming operations; they are in effect sharing the same resources (disk bandwidth)
- log compaction can be incorrectly configured, in which case the rate at which operations are coming in is greater than the compaction rate; thi will result in more and more SSTables being created and we'll eventually run out of disk space
- in a B-tree, a key only exists in a single place


## Column oriented storage

When some of the tables grow very large (tens of columns wide), it is rarely the case querie need to return entire rows. Instead they usually operate on a small set of columns.
In OLTP databases, an entire row is stored as a contiguos set of bytes. This makes retrieving an entire row at once very efficient, but the entire row needs to be loaded in memory and parsed, only for most of it to be discarded later on.

Instead, a column oriented database stores the values in a column together, e.g. in a separate file. Then, for each query, determine which subset of files is actually needed to return a response. 
Each column file stores the entries in the same order, so we can take item N across multiple files and know it belongs to the same row. 

Additionally, it is often the case that values in the same column but across different rows are repetitive. This is a very good use case for additional compression within the column file. For example. we might have 100M transactions, but across 1000 products being sold. Thus, there is a decoupling between the number of rows and the number of unique values. 

Row ordering does not matter, as long as it's the same across column files. Two approaches emerge:
- store the rows in the order in which they are added 
- sort the values within some of the column files. The ordering needs to propagate across all the files, in order to continue relying on the fact that the Nth element in each belongs to the same row. Thus, chosing on which columns to order needs to take into account which types of queries we want to optimize. 
    - by sorting across a column with few distinct values, we improve the compression, as all the repeated values will be next to each other

When writing, it would be very inefficient to insert a row into a sorted table for example, as that would require rewriting ALL the column files.
Instead, we can use an approach similar to LSM trees: store the latest requests in an append log and write them in whatever format we prefer later on, as a backend process (which does not care if it's a row or column based database, the same process applies).

# Replication
---

Reasons for replication:
- to make data more durables: if it's stored in just one location, that machine might fail
- to increase availability: we want the system to continue working (ideally without the user noticing) even in the face of hardware or network failures
- we want the system to be performant, so the data should live closer to the user when possible 

Three main approaches:
- **Single leader** replication
    - one node is elected leader and process all the writes
    - all the other nodes only process reads
    - when a write request arrives, the follower nodes can redirect it to the leader
    - in effect, the writes are serliazed
    - the leader can return OK:
        - as soon as it finishes, leading to eventual consistency (the read replicas are out of sync for a while, shorter or longer)
        - once a majority of replicas have commited the message (two-phase commit)
    - we need a mechanism for leader election in case it fails, see the section on Raft
- **Multi leader** replication
    - more than one node accepts writes
    - offers more availability, at the price of consistency
    - very hard to detect / fix concurrent writes, no silver bullet here
    - data can be lost when deciding which version to keep
- **Leaderless** replication
    - clients send each read (*r*) and write(*w*) to all(*n*) nodes
    - to guarantee reading up to date data, we need R + W > N (quorum consistency)
        - for data that is very often read, set the R to 1 and the W to N - this means we return success only after writing to ALL replicas (with the relevant drawbacks when one goes down etc), while the reads are very fast
        - for data that is often written, set W = 1 and R = N - this results in quick writes, but reading can be slow
        - setting R + W > N guarantees at least one of the values we read is up to date
        - some versioning mechanism is also required; this allows us to determine which value is the most up to date one if several are returned
    - data can be reconciled on reads:
        - when we see some replicas as being out of date as part of a read, set them to the right value; this has the drawback of leaving data as inconsistent for a long while for systems where some values are not read frequently
    - vector clocks can be used to *try* to reconcile stale data; although the work most of the time, there are scenarios where we simply cannot automaticaly tell when an operation happened *before* another and manual intervention is required.

Apparoches to replication lag:
- read-after-writes: 
    - users should always see data they submitted themselves
    - for example: read user settings (tweets, etc.) for the current user from the replica that takes its writes; this ensures that if a change was done and is not replicated, we see it
    - works when writes are deterministically split around replicas (e.g. you can only post changes to your profile on a single instance). 
- monotonic reads:
    - users should not see data jumping back and forth in time
    - if all the queries for a user go to the same instance, this cannot happen

## CRDTs

*Information and images taken from https://www.youtube.com/watch?v=PMVBuMK_pJY*

Represent a family of structures that can be concurrently edited by multiple users. 
These span various data types, such as dictionaries, sorted sets, lists, counters etc. Equivalent to ConcurrentCollections in .NET.
Based on the classification above, this is a case of leaderless replcation: all replicas take writes and try to reconcile them at a later point in the future.

Uses: online chat, shared file editing, collaboration software (Miro) etc. 

**Examples: Redis, Riak**.

The whole idea is for the different versions of a document (text, tasks etc.) to *converge*. That is, for the changes to resolve to the same version of the file across the different clients, while reflecting all the edits that have been made.

An important distinction is whether the users can edit the document offline. If so, the set of changes that need to be reconciled can be very large (think a large change on a git branch). The larger the change set, the more difficult to merge them without conflicts

Precursor: Operational Transform
- translates every client operation into an insert or delete, **based on the index** in the file (character index)
- the server then reconciles and translates operations receied from other clients. If, for example we insert at position 3 and another remote client inserts at position **5** originally, we'll get an update for inserting at positon **6**, which takes into account both operations
- all communications need to go through a central server that reconciles operations

Implementaition:
- each character gets a unique identifier
- assign each character an increasing fractional value between 0 (start of the document) and 1 (end of the document)
- inserting a character between two other ones simply boils down to chosing a rational number between the two
    - when inserting at different positons, it works out of the box
    - however, we might get a mixture of the changes when they overlap in position.
    - when generating random numbers for each positon, we can end up with interleaved text

![crdt interleaved](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/crdt_interleaved.png)

# Partitioning

The main reason for partitioning data is for scalability. Different partitons can be placed on separated *share-nothing* nodes. This allows us to distribute both the data and the queries across different machines, increasing throughput.

Paritioning is usually combined with [Replication](#replication). The data belongs to a single node, but it is replicated across others for faul tolerance. 

## Partitioning key-value data

When deciding how to split the data, we ideally want to make it as even as possible (assuming the nodes all have the same capacity). Otherwise, the workload will be skewed.

### Partitioning by key

Similarly to an encyclopedia, which groups topics by their first letter in volumes, we can assign a continuous range of keys to a partition. Thus, each partition has a minimum and a maximum and we can easily tell which partition contains a certain piece of data the query is looking for.

Furthermore, we can store the data inside partitions in an ordered fashion. Thus, searcing inside them will be fast. This is similar to an [SSTable / LSM-tree](#sstables-and-lsm-trees). 

### Partitioning by hash of the key

We need a function that will return an even distribution across the partition and, furthermore, return the same value for the same key on every call. This is different than how C#/Java do it, where different proceeses will give a different hashCode for the same object. 

Each partition takes an interval of hashes, resuling in something similar to consistent hashing. 
One downside is the fact that adjacent values are now spread across partitions, as we have no guarantee for the hashed values being adjacent as well. In effect, range queries now need to be sent to all the partitions. This is opposed to partitioning by key, where we would know exactly in which partition(s) the results might reside.

A solution is to store data that we want to easily query in the same partiton. For example, we store all the updates from a user within the same partition, e.g. (user_id, post_timestamp). This way, we need to scan a single partition when looking for posts from a certain range.

## Seconday indexes

Once the data is split into partitions, we can support fast queries related to the key on which it was split. If we want to add seconday indexes, we also need to track data in a different format. 
For example, if our objects are cars, we partition by ID and have cars with different colors spread out across the partitions. To allow querying for color, we need to build a structure that will tell us which car IDs correspond to the current query.

This dictionary needs to be split across partitions as well. There are two ways of structuring it:
- each partition stores a representation of the car colors inside it.  
    - when adding a new car, it is cheap to keep this seconday index up to date as all the data lives on the same node
    - when querying, we need to look at ALL the nodes, as any of them might contain cars with the current color
    - this is called a **local index** or **document-partitioned index**
- each partition stores a part of a global index
    - say partition 1 stores references to all red cars, while partition 2 stores references to all silver cars
    - inserting is very expensive, as a write might be spread across different nodes. This is exacerbated when we're building multiple indexes
    - queries are very fast, as we're only looking at a single partition to get all red cars
    - this is called a **global index** or **term-partitioned index**.
    - updates to a global secondary index are usually async, as the change needs to propagate to the other nodes

# Transactions
---

Traditionally described as ACID, it boils down to two main properties:
    - atomicity: when multiple rows need to be changed, either all of them are changed, or none
    - isolation: an operation recived while other transactions are running will not see their data
    - consistency doesn't really mean anything
    - durbility is evident - data needs to be persisted; this is also relative, as nothing is really "durable" - an entire datacenter can be wiped off, harddisks lose information eventually etc

To determine which operations are part of the same transaction, boundary messages need to be included (e.g. BEGIN TRANSACTION and COMMIT)s

## Snapshot isolation

The idea is for every transaction to read from a *consistent* view of the database. This is essential for longer runnng operations, for example backups - we want to back up the state of the database at a certain point in time, with no partial modifications (which can be done wile the backup query executes).

Internal implementation:
- each transaction gets an increasing ID
- the databse keeps multiple copies of the same object in its representation
- each update results in a delete and create; in effect, version N is marked as deleted by transaction X and version N + 1 is marked as created

Ensuring consistency:
- when a transaction comes in, the DB makes a list of all the other executing transactions
- any writes made by transactions in the list are ignored, even if they are applied by the time the current transaction would read. Thus, simply reading all updates made by IDs smaller than the current one would leave us with edge cases

Updating indexes:
- for systems that use [B-trees](#b-trees), we need to update the values from the current leaf up to the parent
- instead of overwriting the values, we simply create a new copy that is modified before being written
- the other unmodified pahts through the tree are left as immutable
- by keeping the copies, we also have a versioning system for the index; every version of the database has its own root, which can be used to get a consistent view of the database at a point in time

## Atomic writes
Ideally the database would offer built in support for some of them. For example, incrementing a value by 1 (similar to Interlocked.Increment in C#).

Some operations have better built in support, where we can manually lock a set of results being returned by a query. For example, if we want to perform a write operation based on the count of a previous read, we mark the set being returned as *FOR UPDATE*, which should lock other transactions from modifying it until we are done.

## Serializability

One fairly recent way of avoiding all the pitfalls of concurrency is to just serialize all write operations in a single thread. If the entire working set is in memory, this operation is very fast; 
Alternatively, we can shard the data into independent partitions and have each shard with its own serialized writer. 

If the types of queries are known in advance, they can be persisted as Stored Procedures inside the DB, which should make each of them faster to execute (and increase the write throughput of the overall system)

## Two-Phased Locking

Normally we want writes and reads to not block each other. Thus, locks are acquired when we want to change the data, not when reading. This however requires snapshot isolation, so that other operations don't see stale data. 
Several reads can go through simultaneously, as long as nobody wants to write to the row. When that happens, we acquire an exclusive lock and block all other reads as well. This prevents them from getting partially modified data. 

Implementation:
- each object in the database has its own lock
- the lock can either be in **shared mode** or **exclusve mode**
- shared mode mdeans multiple *read* transactions can acquire it at the same time; however, they must wait if another has it in exclusive mode
- when a write transaction comes in, it tries to acquire the lock exclusively - **no other** transaction can hold it at the same time, either exclusively or shared. Otherwise, the current transaction must wait. 
- once a lock is acquired by a transaction, **it is held until complete or aborted**.  

The database automatically detects deadlocks and aborts one of the transactions in order to resolve it.

## Optimistic locking

Both Serialization and 2PL are pesmisstic algorithms - they offer a strong guarantee by taking the worst case for every transaction. While this results in no conflicts, it is also slow.

Another aproach is to allow multiple transactions to go through, and check if another conflict one happened before committing. If so, the latter one is simply aborted (and retried).


# Problems with distributed systems
---

Locally running processes work under faily well defined conditons. Additionally, when something goes wrong, the application is allowed to fail (e.g. blue screen) rather than give back bad results. 

As opposed to this, it is inevitable to encounter failures (and do so oftely) in a distributed system. 

## Main causes

- Separate nodes talk to each other over a network that offers few guarantees by design (ethernet). We can never know for sure if a packet will arrive at a destination and if we'll get them message back. 
    - Packet routing can not just lead to messages/responses being lost, but it can take a nondeterministic amount of time to get an answer (during which we migth wrongly declare nodes as dead). Additionally, we can never tell if it's a problem with the receiver, or perhaps the current node has been taken offline. 

- Clocks can never be trusted
    - time of day clocks are usually kept in sync via NPT (a set of servers). This can force a node to set its clock to a time in the past if it's running too fast, resulting in events arriving at the same node to have the wrong chronological order
    - monotonous clocks are better, but they have no meaning outside of the same node

- Processes can be suspended for an arbitrary amount of time at any point, without being able to tell it happened
    - Garbage collection can take a long time
    - the current CPU might be saturated, so it would take a long time to get allocated a new slice, etc. 
    - Operations where we check something time based are very sensitive to this. For example: if we want to do an operation based on how long our lease is on an object, we need to:
        - check the time first
        - perform the operation
        - however, the process might have been suspended for a long time between the two, resulting in a bad state

## Fencing tokens

One workaround to the fact that getting a check against a lease is not enough is to use *fencing tokens*. 
- a server that wants to perform a write operation obtains an increasing token from a lock service
- the token is passed to the storage engine
- the engine rejects operations for which it has seen a larger token
- this prevents a server from accidentally performing a write it shouldn't
- however, the assumption is **there are no malicious servers**. 

![Fencing tokens](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/fencing_tokens.png)

# Distributed consensus
---

## Linearizability

A storage system is linearizable if it *behaves as if we have a single copy of the data*. Some guarantees include:
- that once a write has finished, the new value will be returned to al subsequent reads
- that once a read operation has returned a value, following ones will not return the old one anymore

Some systems when linearizability is essential:
- locking and leader election - all readers must agree whether a lock is taken or not
- uniqueness - in places where a value can only be used once (e.g. unique usernames), the system nodes must all agree if it is taken
- financial transactions - the need for an accout balance not to go below 0
- when data is passed between systems via multiple channels. For example, if we store an object into storage and pass its ID via a queue, it is essential for the receiving system to get it back via storage when querying via the ID. 

![Race condition if not linearizable](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_ddia/linearizable-race-condition.png)

Based on the potential approaches for [replication](#replication), we have the following: 
- **single leader** - potentially linearizable
- **consensus algorithms** - linearizable
- **multi leader** - not linearizable by design, as multiple instances can process different writes at the same time
- **leaderless** - not linearizable, e.g. Dynamo
    - since we only need a partial consensus (usually not all the nodes need to receive the update), we can get stale data on every read

## Zookeeper

TODO - ADD

## Raft

A good visualization here: - https://thesecretlivesofdata.com/raft/

Data replication:
- nodes can be in one of the three states: *Master*, *Follower*, or *Candidate*
    - one node is the master, while the other ones are followers
    - the master receives all write requests

Leader election:
- all nodes start as followers
- once a follower hasn't received a heartbeat / **Append entry** from the master for a variable amount of time, it starts a new election by increasing the term index and voting for itself
- when a node receives a voting request for a term, it votes for the candidate it, unless it already voted in the same term
- the node that gets an absolute majority of the votes is declared the master (so we cannot have more than one)
- if no node gets a majority, they try again. The odds of coliisions are reduced by having variable times for neach node:
    - the *election timeout* represents how long a node waits until it decides to start an election (by becoming a candidate): 150-300ms
- a node will only step down when another node is decleared a leader in a following term, e.g.
    - the current master leads term X
    - due to a network partition, a majority of the other nodes had an election and a new leader was elected (without a majority, they wouldn't have been able to)
    - the initial leader receives an Append entry notification from term X + 1 (or higher), in which case it steps down as a follower


Writing / log replication:
- the values are added to the log, but it is not committed
- the new value is sent to all the followers via the heartbeats
- once a follower receives a write operation, it appends it to its log (still uncommited)
- once the master receives a majority of acknowledgements :
    - it commits the value to its log and returns success
    - notifies the followers they should also commit. This is basically a Two Phase Commit

 # Stream processing
 ---

 Characteristics: 
 - operates on unbound data, that is data arrives gradually and keeps on flowing through the system. This is oppoed to Batch processing, where the size of the data is bound and known at execution time. This is done by artificially dividing the time or data into chunks, e.g. every 1000 records or every day/minute etc. 
 - batch processing has a natural delay: no matter how often we run the process, it will not be in real time. Additionally, as we increase the run frequency, we decrease the amount of data being processed into each interval
 - stream processing aims to process events as they happen, similarly to FileStreams, TCP connections, video streaming etc

## Events

The basis of stream processing is an **event**. This is a small, self contained, immutable object, representing the details of something that happened at some point in time. It usually includes the time of the event (usually the one that writes the event also includes its time).

Event examples: a user took an action (e.g. clicked on a button), a sensor reading was received, a CPU usage measurement was taken etc.

Events are encoded as a descriptionm, so it can be anything serializable - string, json, binary etc. It can be later on stored in various formats: appended to a log, stored in a DB, as well as sent over the network.

Events are written once by a _producer_ and can be read by multiple consumers. Related events (by timestamp, component etc) are grouped in **topics**. 
The consumers can continously poll, but this has diminshing returns; alternatively, they can be notified when a new event is published. The solution is to use a **messaging system**.

## Messaging Systems

Two relevant aspects:
- what happens if events are produced faster than they are consumed? 
    - Messages can be dropped, buffered, or the producer can be slowed down (back pressure).
    - If the messages are stored in a queue, its size can exceed the size of the memory. Do we write them to the disk? How will this affect performance?
- what happens if a node crashes / goes offline? 
    - can messages be lost? Do they need to be persisted as soon as possible? In some scenarios, data arrives regularly and we can afford to lose a few points, e.g. for sensor measurements or heartbeats. 
    - missing a large number of messages might negatively affect metrics, such as counters

### Direct messaging

Examples where no intermediaries are used:
- UDP multicast
- TCP/IP multicast
- direct API calls through REST or RPC, if the customers support it; in effect a webhook, where consumers are notified of new data

In general, the assumption is that both producers and consumers are online all the time. You can implement retries and other similar mechanisms, but the design is not very robust.

### Message brokers

- Represents an intermediary that both the producers and consumers connect to. Clients can come and go, the problem has now moved to the broker itself.
- Some store the events only in memory, while others persist them
- As opposed to direct cals, the system is now asynchronous. Usually the producer gets a success response once the message has been processed by the broker, with few alternatives to be notified when the it reaches the consumers.

Comparison with Databases:
- DBs are persistent, whereas message brokers usually store messages until they are processed
- no "traditional" querying or indexing; however, consumers are notified when there is new data

Message delivery:
- either distributes the messages between the consumers in a RR fashion, acting as a load balancer
- distributes them to ALL the consumers, so multiple systems can process the output
- a mix between the two: distribute them to ALL groups of consumers, while load balancing within the consumers in the same group

Acknowledgements and retries
- brokers wait for the consumers to ACK they received and processed the message; only then it is removed from the queue
    - if a consumer picks it up and processes, but crashes before sending the ACK back, it will result in more-than-once processing, unless other measures are taken
- when redelivery is combined with load balancing, a message can be reassigned to another consumer, but it will be out of order

## Partitioned logs

In AMQP, adding a consumer will only deliver the following messages to it; previous ones are lost.

### Logs for message storage

- an append only log can be used as a message broker: the producer writes at the end of the log, while the consumer reads
- a log can be split into **partitions**, which will reside on separate machines; this allows us to scale, as the partitions are independent
- partitions are grouped into **topics**, which are partitions containing messages of the same type
- within a partition, message ordering is guaranteed, as we only append into a single place; when moving to multiple partitions, some sync mechanism is needed, e.g. vector clocks
- in the same partition, the broker assigns an increasing sequence to messages (the offset)
- can handle millions of messages per second, by partitioning across multiple machines; this provides faul tolerance via replication, and all messages are written to the disk

Compared to traditional messaging:
- fan out is supported out of the box - as any number of consumers can read from a single log, they don't interfere with each other. We can have any number of services reading the log at the same time
- cannot load balance a single partition, as the messages are designed to be read sequentially. Instead, different partitions can be assigned to different consumers / consumer groups, which read the messages in a single threaded manner
- the number of parallelized consumers can be at most the number of partitions; generally a bad idea to have a partition *split* between multiple consumers. Instead, add more partitions.

Consumer offsets:
- each consumer regularly notifies the broker of the last read offset. This is in effect an acknowledgement mechanism, as it means messages with a greater offset have not been seen yet. Due to the serial processing, when a consumer says it has processed message X, it implies X-1 etc have also been processed. Thus, not every message needs to be ACKd.
- the broker is a master, the consumers are followers, similarly to a replicated DB. When a consumer fails, a new one is assigned its partitions, and it receives messages since the last ACKed offset - this can result in duplicate processing, e.g. if the initial node processed but didn't get the chance to send the new offset.
- all the messages are written to the disk, so eventually space will run out. When this happens, older segments are archived or discarted, so the broker is in effect a large **ring-buffer**. This means that it is possible for a consumer to be so far behind, that its offset is before archived / deleted messages. In this case, another mechanism is needed, perhaps to spin a new consumer if the current one falls too far behind. 
- since consumers are independent, this opens up a scenario of consuming the production logs into a test environment, which would not be possible with a regular database
- this is opposed to AMQP, where a message is deleted once it's ACKd. With a log based broker, we don't have such problems

## Databases and streams

A replication log is a stream of database reads and is used for replicating the changes to followers. It is used to keep them in sync, and is very similar to a log based message broker. That is, mulitple consumers applying the same changes will end up in the same state (assuming the changes are deterministic).

When dealing with data stored in multiple places:
- we can have the service update both systems, e.g. the DB and the search index.
    - this raises the problem of atomicity, as transactions are not supported across different products
    - concurent updates might partially overwrite each other, and these are hard to detect
    - one operation might fail, while the other one succeeds, leaving the system in an inconsistent state
    - atomic or two-phase commits might solve the problem, but it's expensive

Change data capture

The solution to the above problem is to introduce a leader and have the other systems follow its writes.   
- the master outputs a stream of changes, ideally as soon as they are written
- the stream is read by the consumers, which in turn apply those changes to their internal state
- as we care about the ordering, a log based system is ideal here: the master appends and the followers read sequentially
- two consumers that have the same offset should have the same view of the data

Log storage

- the current state can be reconstructed by replaying all the events since the beginning of time
- the number of logs can grow too big and firing up a new consumer will be very slow
- snapshots can be used at regular intervals; a snapshot is the corresponding state of the system at a particular offset. New consumers can just load the most rececent one and apply only the following logs
- logs can also be compacted. Instead of haveing to store all the changes to a record, we can store its most current value. E.g. a write of 1 and a write of 3 can be compacted to a single write of 3. Thus, the disk size required is a function of the data set, and not of the number of changes throughout history.
- Kafka supports log compaction, allowing it to be used for persistent storage and not just transient messaging

## Event sourcing

- all the changes to a system are stored as events (the Command part of CQRS). These are immutable user actions and we are storing the action itself, rather than the effect on the database (e.g. overwrite a 1 with a 3)
- updates and deletes are discouraged / forbidden

- the event log does not easily expose the current state of the system. Users expect to see what is in their cart, not a history of it
- thus, the log needs to be parsed into application state which can then be shown to the user (the Query part of CQRS). This means processing it into a different system which is used for reads. 

- as opposed to Change Capture, which contained the entire object, events only represent partial changes (e.g. set updated a field of an object). Thus, log compaction cannot work the same way, as newer events don't necessarily overwrite older ones. 
- the system that stores the data representation needs to take regular snapshots and their associated log offest. This allows a quick reconstruction, but the log itself is still needed.

Commands and Events

- a message broker allows decoupling the time/service that generates an event and the time/service that process it. The whole operation is asynchronous and mostly one way, so a command needs to be validated before being added to the message log. 
- the validation needs to be synchronious with the command itself, that is we would not return a success until the command has been verified. Once success is returned, we should be guaranteed to process it correctly.
- when the message is generated, it becomes an immutable fact. Undoing it will only add another event on top, not modify the initial one.

## State, streams and immutability

Mutable state and an event log are two different views of the same data and are tightly related. The mutable state represents simply the top of the stack, while the event log is the series of actions that took the system to that state. 

- storing the log durably makes it the system of record (source of truth). The database is simply a cache of the most recent values.
- immutable events capture more data: e.g. when adding and removing an item to a cart, the event log will have 2 messages, while storing just the cart state will tell us nothing

- separating the immutable events into a log allows us to have multiple independent systems reading the stream
- by separating the reads, we can build read-optimized views of it and have it run in parallel with the existing system
- regular DBs are slower than an event log due to the fact that we need to update information in random places: on the disk where it's stored, the indexes, etc. If we don't care how the reads are going to process this, it becomes much faster to just append to a log.
- **data does not need to be written in the same format in which it will be consumed**. This simplifies things greatly, as it allows the views to store it however they like on their side. 

Limitations of immutability:

It might not be feasible to store all the changes done since the beginning of time. This depends on what the data looks like.
- if it's a small set that changes often, the stored data will quickly outgrow the size of the current state and immutability might not be the ideal solution
- if the data is mostly adds (e.g. sensor readings), immutability might works nicely
- data might nneed to be deleted for other reasons, e.g. administrative, legal (GDPR). With the log, we cannot really remove the previous entries, we can just make them harder to obtain via application code. Backups are immutable by design, it is not fesible to go and change them


## Processing streams

Once we have the events, we can:
- store them in a different system, e.g. a DB or a search index
- perform an action based on them, such as sending an email or raising a notification towards a user
- process / combine multiple streams and produce an output stream
    - very similar to Map Reduce
    - writes its output in a different location, but also in an append only fashion

Complex event processing (CEP):
- reverses the SQL pattern where we store the data and the query is transient 
- stores the query (think of a rule such as "when more than 20 events happen in X seconds, do something) and tries to match it to incoming data (which is transient)

Stream analytics:
- measuring how often an event happens
- calculating averages over a period (e.g. latency for requests)
- comparing current intervals to previous ones, to determine anomalies, e.g. an increase in resource usage, queue length etc
- Kafka streams (TODO - add more info here)

Querying on streams:
- done similarly to CEP - the query needs to be known in advance, and incoming events are matched to it
- ElasticSearch can be a good candidate for these scenarions

## Time

Two approaches: event execution time, or processing time. 
- Batching relies on event time, as the processing time is irrelenvant
- streaming generally uses the processing time, so it's local

Trying to count events in a window:
- it's impossible to say if we received all the events, or if some are stuck somewhere and will arrive later. This might happen if an instance is slower and the message simply hasn't reached us yet.
- either ingore the straggler events, if they are a small percentage and the business case allows it
- correct the older values when receiving new events. 
- when consuming events from multiple producers, there might not be any sync between their clocks. The only thing we know is sequential is within the same producer. 

## Joining streams
- stream-stream join:
    - due to the possible delays between the two streams, we need to maintain some sort of state
    - need to cache the events from one stream, in order to correlate them with the other stream's data

- stream-table join:
    - *enriches* the events with data from the database
    - need to either look up information on each event, or keep a local copy of the data
    - the local copy can quickly become stale, so we need to subscribe to the changes there as well; in effect, we're storing an up-to-date image of the database (or a subset of it) in the processor

- table-table join:
    - a social network that needs to deliver posts into users' timelines
    - we care about the stream of tweets, as well as a stream of user relationship changes
    - thw stream processor needs to maintain a database of followers for each user so it knowns which timelines it needs to update


The ordering of events is very important in some scenarios, e.g. if you follow -> unfollow vs unfollow -> follow. This is hard to do across partitions, so we should group the events that relate to a user in a single partition. Perhaps userId would be a good way to shard the information to guarante the notificatons end up in the same partition.

## Summary

AMQP brokers (RabbitMQ):
- assigns messages to consumers, which acknowledge once they have processed
- **messages are deleted from the broker once ACKd**
- good as an async version of RPC
- exact order of messages doesn't really matter, and we don't need to go back once a message has been processed

Log-based (Kafka):
- all messages in a partition are assigned to the same consumer group
- messages are always delivered in the same order as they are received
- parallelism is done through multiple partitions
- consumers track their progress via the offset of the last message they processed
- **messages are retained on disk**, so it can be used a store


## Kafka

Use when you need:
- at least once delivery
- message ordering
- replayability

## RabbitMQ
- a message broker, used to deliver information to subscribers/consumers

# Distributed Task Queue
---

Some things to consider when designing a distributed task queue. Notes taken from https://www.youtube.com/watch?v=TTNJKt4I9vg

Collaborative environment
- people need to see each other's pointers for example
- OK to lose soms of the data, as we'll get a new point very soon
- good example for AMQP, as:
    - the data needs to be fanned out to all participants
    - we don't care about ordering

Web crawler - relies heavily on queues for which URLs to process next

Functional requirements:
- decouple the production and the execution of the tasks for a cleaner design

Scheduling:
- add a task, perhaps with a priority
- ask for a task from the queue

Retry policies:
- try 3 times on at least 2 nodes, send to another system if processing still fails

Reoccurring jobs:
- jobs that must be done regularly, e.g. at midnight every night

Mnitoring and observability:
- get the status of a task that was queued
- measure the length of the queue
- measure queue length evolution over time, alert if pressure starts to build up

Durability and availability:
- persist the tasks so they don't get lost in case of failure (use a DB?)
- avoid double processing (use a lock?)
- in practice, these are usually not hard requirements - crawling a page twice or skipping it, probably no loss

Task dependencies:
- do this AFTER this other task has finished (e.g. image/video processing)
- this scheduling should be done in the task scheduler itself
- perhaps a separate service that stores the dependency graph and pushes a task once all its prerequisites are done

Task priorities:
- assume the workers are running at near capacity all the time

Scheduling parameters:
- a task needs certain resources, e.g. run on a particular type of worker

Availability and resilience:
- looking at distributed consensus
- if less than half the machines die, business as usual
- if more than half die, no way to ensure consistency - we don't know if the other set is alive but can't be reached by the master. If it's still alive, it should be the one doing the processing (as it has more nodes). A solution here is to stop accepting writes, but continue serving reads

Exactly once:
- replace "run exactly once" with "have a single side effect". For example, any user facing outputs shouold not be duplicated, such as sending emails, transactioning twice etc.
- require idempotency tokens - don't allow executing the same action twice
    - frontend generates the token and adds it to the request
    - the server marks which tokens have already been seen; if it's a duplicate, ignore the current request
    - downside: the token storage place cannot be sharded
        - is this true? Perhaps the tokens can be generated via a mechanism that will allow sharding, e.g. take the first letter of the user and randomly generate the rest. This would ensure multiple requests from the same user (which have a possibility of being duplicates) end up on the same shard

- fencing tokens:
    - single task should result in a single action being taken; if you need more steps, e.g. send email AND do something else, break it into multiple tasks
    - actions producing a side effect get assigned an increasing token number
    - when the action itself needs to be performed, send the worker the token as well
    - store the token in a durable place, which becomes the source of truth
        - either rght before executing the side effect action, or directly in the other service performing the action, check against the task queue to see if the token is still valid
        - if the operation was reassigned, the database should have been updated and we can avoid performing the action twice (e.g. sending money around)

## Worker API

Relevant points: 
- ease of management for the workers, allowing adding or removing them on the fly (although it's outside of the scope for a distributed task queue)
- large inputs should not be sent via the queue itself; instead we can store them remotely (S3 etc.) and pass a reference to that object
- logs should be stored in a distributed storage system for auditing and debugging purposes (S3)

Push vs Pull:
- if a small amount of tasks that take longer, pull is ok (video processing)
- otherwise, push is needed
- might want to assign tasks in batches
- how does queue know which workers are alive? If a worker dies, you need to redistribute the tasks it took but didn't finish to other instances
    - just because we don't hear from a worker doesn't mean it didn't process; might just be busy and come back later saying it was finished

## Design

- big discrepancy of resources between the task qeuue and the processing nodes
- essential to keep the nodes themselves as busy as possible, i.e. not have one doing the work while the other ones idle

- AMQP can be used for smart distribution to consumers
    - need another layer for persistence, as RabbitMQ is not well suited for that

- Can use Kafka as a source of truth instead of the DB
    - durability and distributed consensus just work