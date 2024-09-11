# Scaling

- horizontal vs vertical
- horizontal is not always the best choice
    - it involves more complexity
    - we should try to scale verically when possible

## Work distribution

- to get the most out of multiple machines, we need to ensure data is distributed **evenly** across them
- simple round-robin is sufficient most of the time, it our requests are stateless and transient

## Data distribution

- distributing work close to where it will be executed
- ideally a node will have all the data it needs to perform its work
    - otherwise, keep the number of nodes it has to talk to to a minimum (scatter-gather)
    - the more nodes, the more likely a request will fail
    - we're vulnerable to long-tail latencies, where 1 request is delaying the entire set

# Indexing

- keeping additional views of the data for fast retrieval
- trades more space (and potentially slower writes) for significantly faster reads
    - no free lunch, all indexing makes writes slower
- in a distributed DB, it will most likely be eventually consistent (GSI in dynamoDb, CDC with Elastic etc)

## Geospatial indexing

## Full-text indexes (Elastic)

- allows lookups based on terms
    - should work for similar words, synnonyms, typos etc

Properties:

- allows searching on documents persisted inside it
- data is sent in JSON format
- can do point(?) queries via GET
- can do range queries via the Search endpoint
    - returns the documents that match the query; can return just a subset of fields if specified in the query


# Communication protocols

- the client and server need to have a common protocol for communication
- most oftenly used is HTTP -> good for request-response usually

Request - Response => HTTP (REST)
Response - Response - Response etc -> Server Sent Events
Bidirectional -> Websockets
    - long polling can be an option => server doesn't respond right away, keeps the connection alive until it has a response


# Core blocks

## Databases

### Relational

- postgres

Main properties:
- good for very structured data
- enforce data type contraints via the column type
- enforce consistency via ForeignKeys (for better or worse, see parsing question)
- more difficult to scale horizontally

- allows any types of queries, i.e. we don't need to optimize for them from the start
- allows long-running, interactive transactions
    - we can read a record, do some actions and write back in the same transaction
    - dynamo only allows sending all the operations in a single transaction

- usually strongly consistent (might not be the case with multiple replicas)

Usecase:

- joins between tables
- great index support:
    - unlimited number
    - allows for multi-column indexes
    - allows for specializezd types, e.g. geospatial or full-text 
- long-running, interactive transactions

### Non-relational

Types:
- key-value (Redis)
- document (dynamo)
- graph (neo4j)
- columnar (cassandra?)

Properties:
- schemaless
- scale horizontally with ease, so can hold more data that a relational one
- data models can evolve without a schema change
    - schema interpretation is passed to the reader

- usually eventually consistent (at least in dynamo's case)
    - dynamo allows configuring it on read

## Blob storage (S3)

Usecase:

- used when we have large amounts of data
- good especially for files we don't need to index or interpret
    - much cheaper to store information than dynamo for example

Properties:

- durable, copies are replicated automatically
- scale to "infinity"
- security features for controlling access to the data, as well as in-flight
- when we control the client, we can bypass the server entirely
    - **pre-signed** URLs allow the server to just generate a URL for the client
    - the client uploads the file directly to S3
- large files should be split into chunks - **multi-part** uploads
    - chunks are ordered to S3 and tagged with metadata
    - once we notify S3 we're done, the files are reconstructed
    - we can upload chunks in parallel
    - more resilient, can retry failed uploads independently
    - **no expiry** => we need to clean up parts ourselves

- S3 glacier => trade access speed for much cheaper storage
    - useful for backups
    - rarely accessed information, e.g. audit records etc


## Search optimized databases

- useful for searching through an amount of text and finding the matches
- regular DBs are not suited, as they require an exact match for a range
    - they scan the entire set normally when querying for a regex (as we can't index for all the terms)

- a search optimized DB breaks the text into tokens and indexes them

- it takes some text and breaks it down into tokens
- we keep indexes for each token, and just add the document reference to the list of references for each token (**an inverted index**)
- additionally, we do some transformations:
    - **stemming** is reducing the words to their root, e.g. running -> run; we also map synnonyms to each other
    - **fuzzy searching** allows us to check words within a certain edit distance; useful for typos

Elastic:

- distributed, scalable
- ingests documents and allows querying for arbitrary strings inside them
- eventually consistent

- a good choice when we want to offer search capabilities, e.g. look up for a product name (with potential typos)

Structure:

- documents are individual entries (i.e. a row)
- a collection of documents is called an index (i.e. a table)

- the documents are partitioned across multiple nodes
- when a range request request comes in, we scatter-gather it:
    - each node has to do a mini-search on its available data
- for a point query, we can use something along the lines of consistent hashing to determine which replica is the leader for that partition
    - we can query the leader directly

Various uses:
- assign weights to different document fields (e.g. headline vs body)
- assign a fuzziness to the query, specifying if we're also looking for 

## API Gateway

- plays the role of routing middleware => sends requests to the right service, based on the url
- offloads some of the cross-cutting non-functional features:
    - authentication
    - SSL termination
    - rate limiting
    - logging

## Load Balancer

- distributes traffic between different instances of the same service
- should allow for different policies, e.g. roundRobin, geographical etc
- we shouldn't care to which instance the request is sent to
    - if we want deterministic directing, use consistent hashing

- can be used right after / in combination with an API gateway

## Queue
## Streams / Event Sourcing
## Distributed Lock
## Distributed Cache
## CDN


# Tools

## Kafka

Should be the go-to for:
- event sourcing
- fanning out, i.e. an event being consumed by multiple services

### Properties

- durable, ensures no messages are lost (via replication)
- scales by adding more instances
- messages are grouped in topics
    - a topic can scale across instance by splitting it into partitions => different partitions can live in different servers

### Messages

- contain a key (used for partitioning), and a payload
- optionally can contain headers and a timestamp

### Partitioning

- a partition is a physical grouping of messages (on the same broker)
- multiple partitions make up a **topic**

1. Default -> key is hashed and we get a partition
- ensures all events with the same key get to the same partition, **as long as the number of partition stays the same**
- can lead to hotspots based on our keys = same problem as partitioning by key
- should not be a problem if the keys are chosen randomly + no particular key is much hotter than the rest

2. Round-robin -> messages get assigned to partitions in a round-robin fashion
- we lose any ordering guarantees between messages with the same key
- we get an even load distribution between partitions => even consumer workload

### Partition replication

- each partition has a partition leader
- it is responsible for all interactions (read/write) with the partion
- the partiton is replicated to other brokers for durability
    - another broker can be promoted as a partition leader if the current one goes down
    - the synchronous replication factor can be configured => kafka only returns success once it has persisted it to N instances

### Consumers

- each partition is assigned to a single consumer => a group cannot scale to more than the partition #

![Kafka consumer assignment](https://raw.githubusercontent.com/strosu/learning-notes/master/notes/images_blocks/kafka_consumers.png)

- we scale the consumers by grouping them in a consumer group
- each event is delivered "once" to each group, to one of the consumers in it
- multiple groups each receive the event "once"

### Brokers

- individual servers (physical or virtual)
- both store information and serve requests
- each hosts a number of **partitions**

- a subset of the brokers are designated as controllers
    - consensus protocol between the controllers
- these handle the metadata storage, partition assignment etc

### Publishing flow

- if a key is specified, hash it to determine the partition to which it belongs
    - otherwise, assign to random / round-robin partition
- determine to which broker the partition is assigned to (via the controllers)
    - we are actually looking for the **partition leader**, which handles all interactions with the replica
- write to broker
- brioker replicates the write to N replicas, where N is the replication factor
    - all the replicas are there just for backup purposes, they don't serve traffic

Messages are stored in an appen-only log file. This has some important properties:

- messages are only appended at the end of the file
- no editing / deleting => messages are immutable
- each partition is represented via its own log file

### Consuming flow

- partitions are assigned to consumers based on their consumer group
- when asking for records from a topic, the consumer will get a batch of them, for its relevant partitions
    - the coordinator keeps track of individual consumers offsets
- when done processing, the consumer sends back an **ack** message with the highest read offset
    - for the next pull operation, the server will only send messages starting with the ack'ed offset

- !!!important to publish the ack at the right time, i.e. when we're sure our work is done
    - our consumer can fail right after sending the ack
- incentive to pubish later => more work for the consumer => more work to redo if it fails
    - thus, we might want to split the work into multiple phases

Parallelism is best achieved by having more partitions.
- this allows us to have consumer groups with multiple parallel consumers
- each has its own offset, and is managed by Kafka
- using multithreading within a single consumer will mess up the offsets we report back
    - we ack only the latest offset, not a list

- the consumer might also keep a list of the last polled offset
- in this case, it will just ask the server for messages starting with an offset, i.e. it will override the defaul behavior

![Kafka partition assignment](https://raw.githubusercontent.com/strosu/learning-notes/master/notes/images_blocks/kafka-partitions.png)

### Scalability

- messages should ideally be small
    - if needed, they can reference larger objects (e.g. stored in S3)
- 1 broker instance can store 1TB of data and handle 10k requests / sec
- we can scale horizontally by adding more brokers
    - however, we need to make sure the topics have enough partitions to have them distributed to the new brokers
    - if a topic has a single partition, it cannot live across different brokers

- we distributed messages across partitions by chosing a different partition key
- too many messages with the same partition key => hotspots / uneven load on the brokers

Hot spot avoidance:

- random distribution of messages
    - if we don't need ordering guarantees, we can leave the distribution to Kafka

- refine the key
    - we can add a suffix to it, e.g. _1, _2 etc => we need to aggregate the results somehow
    - we can further divide it by other dimensions, e.g. geographically => key_eu, key_us
        - useful if we don't need to aggregate over the different sub-partitions


- scalability makes sense **in the context of a topic**
    - different topics are used for different workflows, so they have different scaling requirements

### Fault tolerance

- each partition is replicated to multiple brokers
    - writes are persisted to N replicas before returning (synchronous replication)
    - all traffic served from the leader (thus guaranteeing linearizability)

1. Broker failure
- the coordinator has to promote another broker to be the leader for that partition
- based on the configuration, it can pick any, or just one that has the latest data sync'ed
- a good practice would be to have the **acks** setting to more than 1. Thus we should always have another broker to fall back to in case of leader failure
    - setting **acks=all** would represent the lowest risk of getting an out-of-date broker => max durability


2. Consumer failure

- when a consumer fails, its partitions need to be redistributed to the other ones in the same group
    - should ideally use consistent hashing, so we don't need to rebalance other partitions already assigned to healthy consumers
- consumers from the same group will continue from the last ack'ed offset (stored by the coordinator)


### Message retries

Kafka does not support retries, as it's not a queue. 
- a message queue does not care about how the messages are consumed
- should avoid republishing a message that we failed to process. 

**SQS** is better suited if we want retries, DLQs etc. 
- we can emulate it by creating a retry / dlq topic, but it's not very efficient

### Idempotency

1. On the producer side:

- producers have built-in retries if they don't get a confirmation from Kafka
- can be configured to be idempotent:
    - each producer has an ID and assigns each message a numeric ID (monotonioiusly increasing)
    - the server takes care of deduplication

2. On the consumer side:

- need to consider when the consumer is configured to ack
    - can be as soon as the message is received, and not after it's processed
- best practice is to let the consumers deduplicate messages
    - can be done via a messageId


### Performance considerations

1. Batching

- we can cache writes on the producer and send them in a batch
- can configure it with a time / size cutoff
    - trades performance for a chance of losing the writes


2. Compression

- uses Kafka's built in mechanisms
    - makes the messages smaller => more throughput

- can also consider enconding the messages (e.g. via Avro)
    - more space efficient than JSON
    - producer and consumer need to agree on schemas and versions of it


### Retention policies

- can be configured at an age, or disk space level
- both can be increased, but larger log sizes will degrade performance
- default is 7 days

- can also enable compaction => only the latest value for a messageId is kept
    - same as LSM trees

## Redis

Properties:

- in memory store (although some derivates might offer durability guarantees)
- can be considered to not be very durable
- very highly performant
- each server is single threaded => serializable and linearizable operations

- all data is stored in keys (values, complex objects etc)
  - in effect, the key is the primary key that uniquely identifies an item

### Setups

Has different behavior in terms of replication / partitioning, based on configuration.
**Considerents**:
- all data for a request has to be on a single node
- we should hit the right node directly, not rely on being redirected



1. Single node
   
- single point of failure
- should never use outside of test environments
- one node that handles all read/write operations
  - no partitioning
  - no replication

2. Replicated
   
- introduces the idea of replication between a leader and several followers
- single leader replication
  - a channel is maintaned between the leader and a follower
  - the leader continuously pushes updates


3. Cluster



## Redis Pub/Sub

Properties:

- allows creating dynamic channels on the fly
- much more flexible than Kafka, can scale to tens of millions of channels
- low footprint CPU/RAM wise for an inactive channel
- the only peristed information is the list of subscribers to each channel
- messages are not persisted, so no overhead
    - thus, AT-MOST ONCE
    - messages are only delievered to the consumers that are connected when the message is received by Redis
- messages are delivered in the order in which they were received


Flow:

- consumer A sends a request to subscribe to channel X
- consumer B sends a request to subscribe to channel X
- consumer C publishes on channel X
    - A and B are notified of the message and channel on which it was pushed

- while subscribed, the clients consuming cannot be used for any other purpose
    - we'll need separate clients if we also want to publish


## Redis sorted sets

Properties:

- each member is associated with a value
- members must be unique (e.g. username), but the value can repeat itself (e.g. user's score)
- the value is used to rank the members is an ascending order; two members with the same value will be ordered lexicographically 
- useful for a leaderboard, or a rate limiter
- represented by a hashset (mapping a member to a value) and a **skip-list**, which maps values to members

**Skip list**

![Skip-list](https://raw.githubusercontent.com/strosu/learning-notes/master/notes/images_blocks/skip_list.JPG)

- a linked list that stores the elements in order
    - adding and removing is an O(1) operation
    - indexing is an O(n) operation
- to improve the list traversal, we keep a list containing roughly every other elementand its links
    - at ech step, compare to the next
        - if smaller than the value we're looking for, move pointer to the right
        - otherwise, move pointer "down" (to a lower level)
- keeping multiple layers of lists, until we reach to one that has a single element
    - in total, approx 2N nodes (N + N/2 + N/4 + .. + 1)
    - log(n) layers of lists
- acts just like binary search when looking for an element, as we can always "halve" the next interval by moving right

Properties:
    - O(log(n)) for finding an element
    - O(log(n)) for insertion, deletion of a random element (need to find it, and update up to log(N) other lists upwards, e.g. if we update the middle)
    - the elements are stored in order by design, so getting range of M elements starting from a particular one is O(log(N) + M)
        - we find the element in O(log(N)) and then iterate for another M elements on the lowest level of the list
