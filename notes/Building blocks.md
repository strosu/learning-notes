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
## API Gateway
## Load Balancer
## Queue
## Streams / Event Sourcing
## Distributed Lock
## Distributed Cache
## CDN


# Tools

## Redis Pub/Sub

Properties:

- allows creating dynamic channels on the fly
- much more flexible than Kafka, can scale to tens of millions of channels
- low footprint CPU/RAM wise for an inactive channel
- the only peristed information is the list of subscribers to each channel
- messages are not persisted, so no overhead
    - thus, AT-MOST ONCE
- messages are delivered in the order in which they were received


Flow:

- create a channel A
- user B sends a request to subscribe to channel A
- when a message is pushed to channel A, redis will notify user B


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
