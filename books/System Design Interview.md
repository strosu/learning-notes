# System Design Interview


# Chapter 4 - Rate limiter

- prevents resource starvation via DDoS, by blocking excessive user calls
- reduces cost for expensive APIs - each API has a cost associated with it

## Clarify requirements
- on what should we base the rate limiting?
  - IP is not a good option
  - can rely on userID (requires authentication)
- do we want to notify the callers they are throttled?

Functional requirements:
- accurate rate limiting
- distributed - we want a global rate limit, not a local one

Non-functional requirements:
- low latency, as this will be called for each request
- high fault tolerance

## High level design

Where could the rate limiter live?
- client side: easy to side step, we might not have control over it
- server side
- in between, i.e. a separate layer (in an API gateway)
  - offers a nice separation of concerns
  - more limited, as we don't have full control of the algorithm - are the provided ones good enough?
  - if we already have an API Gateway, we can just tack this on

### Rate limiting algorithms

1. Token bucket
- simplest

Approach:
- each partition/user/etc has a max number of tokens
- each request coming in decrements the counter
  - reaches 0 => reject the request
- at predefined intervals, increment the counter with the refil value
  - any overflow gets lost


2. Leaking bucket
- has a max size each partition can reach
  - this is our burst capacity
- counter starts at 0
- each request can be:
  - allowed, so we increment the counter
  - rejected, if the counter is greater than the max
- at regular intervals, we decrement the counter


3. Fixed window counter

4. Sliding window log
- for each partition, we keep a list of previous timestamps
- when a new request comes in, we remove the timestamps older than now() - window size
- if the set's count is less than max, allow the requets

Cons:
- we need a lot more memory 
  - timestamps consume more than counters
  - we need to store several timestamps per partition
  - we also store failed requests (TODO - clarify the atomicity complications here and why this is required)

5. Sliding window counter
- have two counters, one for the current window, one for the previous
- when a request comes in, compute an average:
  - if we're 1/3 in the current window, take 2/3 of the previous counter, plus the current one
  - if within the bounds, allow and increment current
  
Pros:
- much more efficient memory-wise, as we just have two counters per partition
- we don't count rejected requests (do it all the Lua level in Redis)

Cons:
- it's an approximation, but a very good one (Cloudflare example)

![Rate limiter](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/rate-limiter.png)


# Chapter 5 - Consistent hashing

- The goal is to load balance partitions
- keys should be evenly distributed amongst partitions
- Adding or removing one should cause miniumum rebalancing

- We need a hashing function with a large space, e.g. Sha-1: [0, 2^160]
- connect both ends to get a hash ring:


![Hash ring](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/hash-ring.png)

- using the same function, we hash the servers (by IP, name, etc)
- assuming enough servers, the distribution should be even around the ring
  - additionally, we can use virtual nodes => each physical server has multiple values hashed on the ring

To assign a value to a server, we:
1. hash the value
2. walk clockwise from the hashed value until we find a server
- binay search might work better, as we know the intervals ordered by their starting point
- takes logN to find the placement, where N is the number of intervals


# Chapter 6 - Key-value store

Any data storage has to chose between Consistency and Availability

Consistent:
- will return an error during a network partition
- will always guarantee to return the latest data
- similar to a synchronous replication

Available:
- will continue to work during network partitions
- does not guarantee to always return the latest
- more performant, more robust but is not suitable for some scenarios (e.g. bank accounts)

Essential aspects:

1. Data partitioning

- the data won't fit into a single node
- we need to split it across multiple instances, with minimum movement on node addition / removal
- Consistent hashing is a good solution
  

2. Data replication

- we need to store multiple copies of each item
- when using consistent hashing to determine which partition holds the leader copy
- we can follow up from that node until we find N distinct **real** nodes and copy it there too
- we need to determine a consistency level: strong or weak (eventual)

Suggested approach is to use leaderless replication, which will result in conflicts
- we can fine-tune the performance based on quorum (W + R > N)
- if we can have our storage live in a single location (e.g. how dynamo and azure do), perhaps multi-leader replication would be better suited
  - this would prevent conflicts, as each replica is responsible for its own subset of reads/writes

3. Failure detection

- multicast would work for a small number of nodes (all-to-all) communication
- for a larger network, **a gossip protocol** can be used
  - each node keeps a list of heartbeats from all others
  - each node talks to a random subset of its peers
    - when one of its gossipers is down, it forwards this information to the other nodes it talks to
    - these nodes update their information and propagate it
    - if a majority agrees that a node is down, it is marked as such
  - no centralized authority to detect a node is down => much more robust

![Gossip protocol](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/gossip-protocol.png)

Temporary failures:
- a sloppy quorum can be used initially (+ hinted handoff when the node is back)
- how do we transition from a temporary to a permanent failure?
  - we already store the information on the healthy nodes, so when the partitions are "moved", the data for the sloppy quorum would already be there
  
4. Data integrity

- we don't want to have to compare the entire set, so a regular checksum is not enough
- instead, the data is split into buckets - groups of keys and their values
  - each non-leaf node holds a checksum for its two children
  - if the root nodes are equal, the trees are the same
  - if not, dig into it until we find the bad bucket

![Merkle tree](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/merkle-tree.png)


5. Read / write paths

- we can rely on either SSTables or B-Trees for the storage part

SSTables:
- writes go to the WAL / memtable
- WAL gets flushed regularly (however, not on each operation)
- memtable gets flushed to an SSTable when the size is reached
  - each SSTable continues to have an in-memory sparse index (holding just some of the keys and their offsets)
- we need to do compaction (similar to a garbage collector)
  - different approaches for this, one of them is generational (just like .net does it)
  - keys are sorted in the SSTables, easy to pass them through a streaming merge-sort algorithm
  - 

- reading checks the Memtable first
  - if found, it's the latest value
  - if not, search through the SSTables from the newest to the oldest
  - a bloom filter can be used to eliminate certain SSTables completely


# Chapter 7 - Unique ID Generator

## Requirements

- IDs must be unique
- IDs must be sortable (so we can't just use GUID)
- length requirement -> max 64 bits
- large number of generations / second, >10k


### UUID approach

- easist to implement, as generation is independent of the multiple servers
- it requires 128 bits, so it's too large
- uuids also contain letters, so it wouldn't work
- more importantly, they offer no ordering

### Snowflake approach

- we divide the 64 bits into chunks
- as we need the IDs to be sortable, the first part should be an incrementing timestamp (counting milliseconds)
  - how many bits do we reserve for it? 41 seems to allow for a large enough range (69 years)

- this assumes timestamps are always ascending, which is not the case
  - we get **some** ordering within the same server (excluding the clocks jumping backwards)
  - messages generated across different instances might be out of order, due to clock skews

- the second part is used to differentiate between machines (DC + machine ID etc)
  - 10 bits allow for 1024 different instances, which should be enough

- the last bits are a counter pe millisecond (reset to 0 every ms)
  - using 12 bits would give us 4k unique IDs per instance
  - we can tune this based on the number of instance we currently have, their capacity etc
  - maybe we want less variance in the 2nd part of the ID and more in the latter

![Snowlflake IDs](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/snowflake-id.png)

# Chapter 8 - URL shortner

- needs to support two operations:
  - take a URL and create a short key for it
  - given a short key, retrieve the original URL

Clarify requirements:
- 100M writes / day => 1200 writes / sec, with 2500 writes / sec burst
  - max entity size: 100 chars = 100 bytes => 10 GB / day => 3.5TB / year
  - max years of retention? TTL if we want to expire them faster?
  - max number of entities: 100M * 365 * year_count (10) => N max entities
  - we can use a set of chars for the short URL (0-9a-zA-Z) => **62 different chars (M)**
    - do (log(62) N) + 1 => needed number of chars for the short URL (**X = 7** in their example)
- 10x reads => 25 000 / sec => cache required

## Write path

- we consume a string and must return a shorter URL

1. Generate a unique ID and transform it to base M
  - no need to check for collisions, as unique IDs will result in unique urls
  - the length of the url will vary, starts as 1 and grows
  - easy for an attacker do enumerate other short URLs, as the IDs are publicly exposed and incrementing

2. Hash the input URL and take the first X characters
  - might result in conflicts, so we need to check if the value is already in use
    - if the prefix is already used, append some suffix to the input and hash again
    - checking if it's been used would normally result in a trip to the DB
    - we can use a bloom filter to remove some of the unnecessary checks, as most of the time we won't have already used the value
  - the length of the URLs will always be the same
  - does not require a separate service to generate unique IDs

![Hash generation conflict](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/hashing-conflict.png)

- whichever options we chose, we need to store the URL into a key-value store (e.g. DynamoDB)
- when receiving a write request for a value we already shortened, we can return the value directly instead of a 409 for example
  - in effect, it makes the API idempotent

  ## Read path

  - check the cache first
  - if found, return the long URL as a header
    - 301 is a permanent redirect => result gets cached by the client
    - 302 is a temporary one => follow up requests will eventually call our servers again
      - will lead to increased traffic, but also better metrics on the usage of the system

# Chapter 9 - Web crawler

- two main areas of responsability:
  - generate URLs to crawl
  - fetch their content and process
    - can be saving the content for later ingestion
    - can be building an index of the content
    - DRM etc

1. Gererating the URLS (URL dispatcher / frontier)
- we start with a pre-seeded list, based on revelence (e.g. wikipedia, news sites etc)
- need to respect the hosts: robots.txt, as well as politeness (frequency of polling a single domain)

2. Pulling the data
- several worker instances are polling the URL dispatcher for which URL to crawl next
- they resolve the DNS name and retrieve the content
- we cannot rely on the websites being plain HTML, so we need to generate them from dynamic content (js etc)
- either the same worker can do this, or it can be a separate role
- we get the actual content and generate its hash
  - this allows us to discard it we've already seen it (which should be a significant chunk of the data)
  - we store the content in S3 and drop a message for the parser workers
  - we can use a Kafka queue here 
    
 Optimizations:
 - distribute the workers performing the actual GET requests
  - how do you know from where a website will be served?
- aggressive timeout, as some websites will never return

3. Parsing the data
- once we've retrieve the data (via the refefence from kafka), we can have multiple consumers:
  - one the parse the URLs for the next iteration
  - one to parse the content for LLM
  - build an inverted index etc.

- parsing the URLs:
  - we check we haven't already seen the URLs, we the graph contains cycles
  - we perform additional checks based on the URL, e.g. to cap its depth

  ![Web crawler](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/web-crawler.png)

  
# Chapter 10 - Notification system

- multiple types of notifications
    - SMS
    - push notifications: differentiate between iOS and Android
    - email

- need to offer an opt-out mechanism (GDPR etc)
- we use various services for each type of notification:

1. For mobile, both Apple and Google offer their intermediary service we have to go through
  - we require a device identifier, which can be obtained after getting consent from the user
2. For SMS, something like Twilio
3. For email, MailChimp etc

Two workflows:

1. Gathering contact information
  - device info vs email vs phone number
  - this is the metadata, store in a relational DB probably. Some document store can also work, as there's little correlation between different users

  - we can have a users table + a device table, with a simple join between them
  - need to have the correlation, as the notification shouold be addressed to a user -> might have to fan out to multiple devices

2. Send the actual notifications

- our server receives a request to send a notification to user A
- it looks up which types of notifications this needs to result in (e.g. just push notifications vs email etc)
  - the requester might also be able to specify this

- we want to allow the notification system to be pluggable, e.g. to support new types of notifications easily
- a simple approach is to decouple the actual sending logic from the rest
  - for each type of notification we support, we have a set of workers
  - they get requests from a queue and act on them
  - offers good decoupling between deciding what notifications to send out vs the actual sending logic
  - offers good scalability, as we can adjust the capacity based on throughput * avg time to process a request (e.g. email might take longer etc)

- we also provide a queue for each set of workers
- thus, the notification service is responsible for preparing the needed types of notifications and dropping them in the queue
- it can then return to the caller with a promise that the request will "eventually" succeed

- the service needs to retrieve the information we persisted in step 1
  - we can cache the user / device information if it's not too large

### What type of queue should we use?

- since we need retries and don't want to block, Kafka might not be the best candidate
  - for Kafka, we can't acknowledge follow up messages until we process the current one
- look into AWS SQS 
  - offers exponential backoff retries 
  - can hide a message so we process the rest in the queue

## Deep dive

1. Reliability
- we should never lose data
- the ordering is not the most important aspect

- pushing to a queue is not sufficient?
  - we can always store the notification request in a DB and only return to the caller once that's been done

2. Exactly-once delivery
- is probably not achievable
- we can always lose the successful response from the 3rd party
  - we have to retry in those cases, as we don't know the outcome
  - unless the 3rd party offers some idempotency mechanism, we will deliver messages more than once

3. Notification templates

- we should have a small set of templates, relative to the number of requests
- the callers can specify a type of notification, together with the request information (content, recipient)
- no need to send the entire body of the message each time
  - additionally, the senders don't need to know the final format of the messages
- we have to store these somewhere
  - both persistent storage
  - a cache would be a good fit, since they shouldn't occupy too much (as it doesn't scale with the number of users or notifications)

4. Notification settings

- before sending any notification, we need to make sure the user gave their consent
- this can be per channel if needed
- this also touches on rate limiting, as we don't want to spam the users

5. Sending failure and retries

- the workers are trying to perform the notification synchronously
- if the flow succeeds, we remove the nofication from the queue
- otherwise, leave it there (with an exponential backoff visibility)
- SQS gives us information on the number of performed retries
  - we can then move it to a DLQ and alert the oncall person

6. Monitoring

- we need to strictly monitor the amount of notifications waiting to be processed
- we need to alert when it reaches a particular threshold
  - this is so we can adapt the number of workers (add or remove some)
- additionally, we need monitoring when a notification fails to be delivered multiple times

7. Analytics

- we can track each notification's lifecycle
  - created
  - delivered
  - opened / acted upon / unsubscribed
- use a state machine for easier implementation 

![Merkle tree](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/notification-service.png)


# Chapter 11 - News feed

Two main flows:
- publishing content
- retrieving the feed for a particular user

Overall concerns:
- we only support authenticated calls
- we use a rate limiter to prevent abuse
- all of these (and more) can live in an API gateway
  - load balancing
  - SSL termination etc

## Publishing a post

What happens when a user creates a post?

1. Analyse it for malicious / illegal content

2. Persist the post
  - store the large pieces of data in S3
  - store the post itself in a DB
  - cache the result? Why?

3. Insert it into other news feeds (if we want to)
  - what do we store here? Just a postId
  - how do we store a news feed?
  - book suggests keeping it all in caches for fast retrieval
    - it this fesible? It can be partitioned well, so we should be able to scale out
    - we can also rebuild these in case they are lost?

Two approaches:

- prepare the feeds when a new post is created (fanout on write)
  - the feed is pre-computed and is pushed to friends immediatelly
  - reading is fast, since it's already calculated
  - useful if we expect the feed to actually be consumed; not the case for rarely logging in users

  - updating all the different feeds can be resource consuming


- prepare the feeds when a user asks for them (fanout on read)
  - does not waste resources for users logging in rarely (e.g. compute it once a month)
  - fetching is slow, as it has to be calculated just in time

Normally we place a lot of value on retrieving the feed fast, so it's best to prepare most feeds on write.
Depending on the max amount of connections, it might be impractical to do this for all users. 
- we can define a threshold: if a message has to be delivered to more than N feeds, we do it on read (e.g. Obama posting smth on facebook etc)
- this is definitely needed when there's no upper bound, might work if all users are limited to X friends (although bad for business perhaps)

Concrete steps:

a. fanout service gets the list of friends of the current user (so we know which feeds to update)
  - we can use a graphDB or similar as an efficient model for this

b. we need to determine if a message should be delivered to a particular friend
  - users might be able to mute each other, or post a message to only a subset of friends
  - determine the subset of users whose feeds have to be updated

c. enqueue a command for the feed population worker
  - either one notification for the entire list of friends (i.e. batches), or one by one 

d. the fanout worker / feed population worker inserts information in the feed caches
  - for each user, store a list of the most recent N posts (just via their ID)
  - it should be ok to be someowhat slower to retrieve posts from years ago, i.e. not cached ones

[INSERT POST IMAGE]

4. Notify other users

- should have the same verification logic as the fanout service


## Retrieving a feed

- most of the time, the feed will be prepopulated
- we also need to take into account users with a heavy following, for which we don't insert at write time
- we append any recent messages from vip users into the computed feed


# Chapter 12 - Messaging service

- should allow sending messages securely from user A to user B if both are online
- if a user is not online, it should deliver those messages once they come back
- users can start group chats with up to X users 
- media can be part of the messages

- how long do we need to store the message history server side?

Non-functional:

- deliver messages with a low latency
- if a message is accepted by the server, guarantee its eventual delivery
- scale to a large amount of users
- highly available system
- messages should not be stored longer than necessary
- security / spam prevention / content analysis

# Basic design

- we can identify a 1-1 chat as a case of a group chat with 2 participants
- the whole system can be seen as delivering messages from a user to a group, and from the group to its users

The communication between the client and server is bi-directional. Thus, we can use **websockets**.

Types of messages the client initiate:

- createChat
- modifyParticipants

- sendMessage
- addAttachment
- ackMessage

Types of messages the server initiates:

- newMessageReceived
- chatParticipantsUpdated