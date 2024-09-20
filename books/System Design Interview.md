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

## Basic design

- we can identify a 1-1 chat as a case of a group chat with 2 participants
- the whole system can be seen as delivering messages from a user to a group, and from the group to its users

The communication between the client and server is bi-directional. Thus, we can use **websockets**.

Types of messages the client initiate:

- createChat(participants: List<String>, name: String) -> chatId: String
- modifyParticipants(chatId: String, name: String, operation: Add/Remove): Success/Fail

- sendMessage(chatId: String, message: String, attachments: List): Success/Fail
- addAttachment
- ackMessage(messageId: String)

Types of messages the server initiates:

- newMessageReceived
- chatParticipantsUpdated


### Attachments

- the servers should not be handing attachments themselves, as they would just be a proxy for S3 anyway
- instead, when at attachment request is received, get a pre-signed URL from S3
- pass the presigned URL to the client, so they can upload directly
  - if we want to support large files, we need to do a multi-part upload
  - this introduces the overehead of cleaning up the incomplete attachments

## Models

We need to persist information about:

- users
  - metadata: slowly changing, e.g. name, photo, phone number etc
  - status, if needed. We might want to separate it from the rest of the information, as it's access patterns are different (and this is much more lightweight)
- chats: mapping of what users belong to which chat
- messages themselves
  - this is the message content itself
  - additionally, links to attachments stored in blob storage
- deliveries
  - tuple of (message, client) that needs to be delievered
  - keeps track of when a message is delivered to all the participants, so we can delete it

![Whatsapp](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/whatsapp-redis.jpeg)

## Deep dives

### Scaling 

Chat server:
- the only stateful component is the chat server
- we cannot work around this limitation, as a websocket needs a persistent TCP connection
- when a client connects, the request gets routed to a chat server (RoundRobin, or based on load -> delegated to the load balancer)

- assuming 1M connections per server, we need hundreds to be able to scale 
- we can add more servers to support more concurrent users

- when a client connects, the server subscribes to pub/sub channel for that user
  - we need Redis to keep a connection open to all the chat servers, but the servers themselves don't need to know about each other
- when the websocket connection times out / we don't receive heartbeats, unsubscribe 

### Resilience

- Redis should be running in clustered mode, for scalability and resilience purposes
- the pub/sub channels get sharded across multiple instances, allowing it to scale
  - we might need to add more redis instances if needed, but we can scale horizontally as much as we want / can afford


## Logging in

User logs in:
   1. it sends a known offset for each conversation it is part of
   2. the persistence service queries the conversations table for each of them
      1. our partition key should be the tuple of (chat_id, timestamp / offset)
      2. the sort key offers a natural ordering of messages in a conversation
      3. we do a range query for all entries with a particular partition key (the conversation_id), and an offset / timestamp higher than the one the client already knows about
   3. it returns a list of messages
   4. client applies the changes and acks them
   5. The events table also gets updated, since the messages have now been delivered

User connects to the chat server:

1. The LB creates a persistent websocket connection between the client and the webserver (with it acting as an L4 LB)
2. the chat server subscribes to the pub/sub channel for that user
3. any follow up messages are pushed from the chat server to the client

## Sending a message

1. User is already connected to a chat server
2. The client pushes a message via the websocket
3. The chat server persists the message to durable storage
   - This can be an SQS queue, so it's not directly coupled with the persistence servers
   - sending a message can be done asynchronously, but other operations might not be so straighforward (e.g. adding a participant)
4. The persistence servers can scale independently. They read from the SQS queue and do the following:
     - determine to which participants the message has to be delivered
     - create events for each of them
     - publish a message to Redis for each user that has to get a notification
   

# Chapter 13 - Top K searches

## Basic design

Conceptually, we need to be able to answer a query of "given a preffix, what are the most likely K words searched for?"

- We store a map of search term - count.
- when a request comes in, we need to find the subset of search terms that match it
- out of the subset, pick the top K 


- if we operate on preffixes, a trie is an ideal sollution, as we're constantly narrowing down the scope
- to conclude, an up-to-date trie should be able to offer us the responses very quickly

Access patterns:

- we perform one write, when the user actually presses Enter
- we need to offer multiple intermediate results, potentially with every keystroke. **we might want to send out the requests every interval, e.g. 50ms**

- the system will be heavily skewed towards reads, so we should optimize them over writes

## Querying

- the leaf nodes will be at varying depths in the trie 
  - thus, we need to search all the children nodes of the current preffix, so we can find the top k

- we can precompute some of the results and store them in the intermediary nodes
  - e.g. for T, we store Twitter and Twitch
- downside is that we require multiple updates for each write
  - we don't just need to increment the counter for the node, but also see if its ancestors are still up to date

![Autocomplete trie](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/autocomplete-trie.jpeg)

## Storing the data

- the advantage of a trie is that we can also see its children
  - this might not be too useful, if we also store the data in intermediary nodes
- alternatively, we can flatten the trie into a key-value map
  - we're mapping each preffix to a list of most used for queries
  - for each read operation, we don't need to traverse the tree, instead we do an O(1) to get the value

![Autocomplete trie](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/flattened-trie.jpeg)

## Updating the data

- assuming a large enough scale, we don't want to count every single request
- instead, we can use **sampling**, i.e. we look at 1 of N requests

## Populating the trie

- we need to process a stream of incoming requests:
  - when a user executes the query, have a chance of 1 / N to persist the sample to a kafka topic
  - we need a large amount of consumers, to accomodate processing a large amount of logs in parallel => we need a high number of partitions (thousands? depends on how many writes per second a consumer can handle)
- consumers read events from the kafka partitions and update the relevant rows in dynamoDB
  - this should also update all previous preffix rows, so we need to limit the max depth of our "trie"
- dynamoDB handles the partitioning, we don't need to worry about figuring out which instance holds which data
- as a paranthesis, we should aim to keep our values in the table below one RU/WU from dynamo (1kb)

![Overall design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/autocomplete-overall.jpeg)

## Other discussion points

- as the input is taken directly from the users, we might need to support Unicode characters
  - this will have an impact on the size of our trie, as it significantly expands the number of nodes we need to keep track of

- multiple top-k sets of results
  - the searches are very coupled with the users' language
  - with the current implementation, we represent the top-k searches for a single language
  - we might want to have multiple tries, one for each language we support

- different weights for different searches
  - in the current setup, we assign equal importance to a search from 10 years ago, as to one from today
  - we might want to prioritize more recent searches, as those are more likely more relevant
  - in the current system, we never purge the data


# Chapter 14 - Youtube

## Requirements

- users should be able to upload a file (large size)
  - we should support upload pausing / resuming
- users should be able to play a video

- multiple clients should be supported => multiple encodings of the same file
- clients have different netework bandwidths => multiple quality versions of the same file


Non-functional:

- system should be highly available
- minimize the time needed between the upload and the video being available (e.g. X mins)
- system should support a high throughput (millions of videos per day uploaded, 100s of Ms of views)
- video playback should be smooth, even in low bandwidth environments
- minimize infrastructure costs

User locality: international, distributed across several markets

## Entities

User
- some basic information about the user
- profile picture etc

Video
VideoMetadata

## Video related concepts

1. Codec
  - represents the encoding of the video, so it has a small size
  - e.g. H.264

2. Container
  - represents the storing of the video
  - codec + some metadata
  - e.g. MP4

3. Metadata files
  - contain information about the different tracks
  - contain information about different bitrates
  - can map (format, offset, quality) -> encoded video chunk
  - the client can decide which chunk to get next (requires some logic on the client side)

## Uploading a video

![Overall design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/youtube-initial-upload.png)

- as we want to provide a smooth experience, we can use **adaptive bitrate**
  - this will mix and match the video segments of different quality / bitrate, based on the client's network

Video processing service:

- gets notified a video has been uploaded the S3 by the client
- breaks it into segments, a few seconds each
- each segment is then converted / encoded to different qualities
  - segments are stored in S3
  - we update the metadata for the orginal video, to also include a map of (video, offset/segment/quality) -> S3 url

- this kind of work will be mostly CPU-bound, as it's computationally intensive
  - thus, we want to parallelize as it as much as possible

- different worker pools for different tasks
- shared-nothing between segments => the orchestrator can drop a request for each to be processed in parallel
- if a step is based on a previous one, we need to chain up queues / workers
  - when step A is finished, just drop a message to do B in their respective worker queue
  - the intermediate segments are also saved in S3

- if a processing step fails, don't mark the message in the SQS queue as processed
  - it will get picked up again by another worker, providing a retry mechanism
  - if we decide it's an irrecoverable failure:
    - terminate the process
    - remove temporary pieces

- each step / step chain will update the cache for the processing
  - the orchestrator regularly checks if all the works is finished (this can also be a separate role)

- when all the steps have finished:
  - update the metadata DB with the correct information for the video
- if the process fails / cannot be finished, remove the intermediate segments from S3 and from the cache

![With parallel processing](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/youtube-with-dag.png)


## Viewing a video

- not all videos will have the same number of views
- 80/20 principle, apply recursively -> 64% of views will be towards 4% of the videos
- we add a CDN layer on top of S3, and cache the frequently viewed files
- accessing video metadata is a frequent operation too, so we need to add a caching layer here as well

- the client will get the file metadata first, resulting in a map of (segment / quality) -> s3 url
- based on the type, performance etc. of the client, it decides which rate to use
- retrieves the segment from a cdn / s3 and plays it

## Multipart uploads

- the client breaks the file into chunks
- it posts the list of chunks to the API
  - API persists it to the metadata entry, with a status of Created
- the client starts uploading pieces to S3
  - when a piece is uploaded, have S3 trigger a function to upload the metadata entry for that chunk and mark it as Done

- if the upload is interrupted, the client can query the server and get a list of chunks that are not yet uploaded
- the process resumes

## Other discussion points

### Speeding up the processing

- we can start processing the video segments as soon as a user uploads them
- the S3 trigger can also drop a message for the orchestrator, which will pick it up from there
- there will inevitably be cases when we're left with incomplete uploads
  - set up an upper timebound until we consider it to be abandoned -> this will also reflect in the max amount of time for which an upload is resumable


### Resuming a video view

- we need to introduce another concept, e.g. a session
- this can keep track of the user requesting the video, and the last segment it has viewed
- the client needs to keep pushing this to the server every few seconds
  - we need a persistent connection here
  - we might not care about the precise granularity, i.e. have the client send their offset every 5-10 seconds

- will add significant traffic
  - might want to look at having a separate service whose sole job would be to track this / update the cache
  - the client can query this service and handle the resuming on its own 
    - it would know the list of segments, as well as the last one


![Final design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/youtube-final.png)


# Chapter 14 - Dropbox

## Requirements

1. Functional

- able to upload a file
- the file should sync to all the devices for that user
- able to download the file to any device
- able to see / download files shared with them by other users
- file versioning

2. Non-functional

- highly available
- should support high throughput
- should support large files (GB size)
- upload and download should be as fast as possible
- files should be secured during transport and storage

## Entities

1. File
2. FileMetadata
3. User

## Api

- NOTE: All API requests should be authenticated via a JWT in the header

- POST
  - prepareUpload(metadata: FileMetadata) -> uploadUrl
  - upload(uploadUrl) -> actually against the bucket storage
- GET
  - getFile(fileId) -> File and FileMetadata
- POST
  - shareFile(userSet: List<UserId>)

## Basic design

![Basic dropbox design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/dropbox-basic.jpg)

### Upload path

- user notifies the server it wants to start a file upload (and sends the metadata)
- the upload server:
  - saves the metadata
  - gets a pre-signed URL from S3
  - returns the pre-signed URL to the client
- the client chunks the file and uploads them to S3
- when the upload is finished, trigger a completion against the multi-part file
- a lambda function notifies the upload server, who then updates the metadata to reflect the file has been uploaded
- the server can optionally push a notification towards the other clients so they keep the file in sync

### Download path

- user requests a download URL from the server
  - the file should be replicated across different S3 regions for durability / performance reasons
  - we can also cache the file in a CDN, although the read / write ratio is not ideal for this usage
- the server checks if the user has permissions to access that file
- the user returns a URL, either an S3 or a CND one

### Share a file path

- user that owns the file sends a request to grant access to user A to file B
- we store this information in a dynamo table where A is the partition key and B is a sort key
- when checking if a user has permissions, we simply do a point query for (A, B)
  - if we get a result, means the user has permissions
  - no results means no allowed
- we can also easily check what files a user has permissions to by doing a range query via the partition key for user A

### File changes

- since our file is composed of segments, the client should be able to deduce which ones changed
- we upload only the diffs between the initial file and its new version
- for each version, we store the metadata
  - this will most likely be a mix of chunks from the current version (changed), as well as older ones (not changed)
  - we treat the chunks like a heap file, and just reference them from the metadata

How do we expire versions?
  - we need to define how many versions we store (e.g. last N)
  - we shouldn't save on every single change, instead batch these changes when pushing to the server
  - once a version is above the expiry threshold, remove it, as well as any unreferenced chunks

## Deep dives

### Handling large files

Since we need to support files up to 10GB or more, we cannot use a single request to upload the entire file.
Drawbacks:
  - timeouts on the server / client side
  - network interruptions -> we have to start all over again
  - sequential upload is slow

Solution: 
- break the file into multiple parts
  - requires a "smarter" client
- each part can be indepentely uploaded (in parallel)
- we can pause / resume uploads by keeping track of which chunks have already been received by the server
  - the metadata file can keep more verbose information
  - instead of created / uploaded, we keep a list of chunks and the status for each of them
  - when the blob storage finishes processing one chunk, it notifies us and we mark it as completed
  - resuming implies querying for the metadata, and seeing which chunks have finished uploading

If we want to support multiple versions / users editing the files after the upload:

- we cannot rely on S3 to handle this for us automatically
- emulating the behavior would work:
  - the file is split into pieces from the start
  - each piece is an independent S3 file (not part of a multi-part)
  - the client uploads all of them at the start, with metadata being kept in sync
- when an edit is made, identify which segment(s) changed
  - push metadata information about the new version
  - references all non-changed parts
  - upload the changed parts

- we can keep a map of (segment, oldest version referencing it)
  - this gets updated on every edit, unless the current version has changed that segment
  - when we expire a file version, check for all segments that have its key (use a GSI)

### Performance

- we upload as much of the file in parallel as the client's bandwidth allows 
  - since the chunks are indepent of each other, they can be processed fully in parllel (and this scales horizontally)
- since the bandwidth is probably the bottleneck, we can try and reduce our usage:
  - each chunk is compressed before sending
  - we can also store a compressed version in S3 once the file upload completes
  - the client will then receive a compressed file, that it needs to decompress
- compression efficiency varies greatly based on the type / content of the file
  - use some heuristics to determine if the computational effort to compress the file is worth it

### Security

Encryption in transit:
- https provides it for the communcation between client and server

Encryption at rest:
- S3 provides encryption by default for all the files that it stores
  - getting the file won't be sufficient without the decryption key

Access control:
- when a user requests a file, we check if they have access to it
  - this is done via the permissions table (check for (userA, fileB) exists)
  - if the user has permissions, we generate a pre-signed URL 
  - the URL should be expiring, with an agressive timer (e.g. 5 minutes)

### Performance

- all our services are stateless, so they can scale horizontally without any problems
- we can consider S3 to be "infinitely" scalable, so storing the files there should be fine
- cost-wise, we might want to look at offloading some of our less used files into cold storage for cost reduction purposes
  - the read-write ration is about a 1:1

### File update conflicts

- if multiple users have write permissions to the same file, we might run into concurrent updates

Problems:
  - we have no clear way of differentiating which update should take precedence
  - merging the updates makes little sense

Solution:
  - we store both versions of the file and let the users sort it out
  - could three way merge be used to solve the problem? It still requires user interaction for conflict resolution


![Dropbox detailed design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/dropbox-detailed.jpg)


# Part 2 - Chapter 1 - Proximity service

Used to discover nearby points of interests (restaurants etc)

The points of interest are static, so this would not fit a "find my friends" scenario.

## Requirements

Three main operations:
- add a point of interest
- get nearby points of interest
  - should support multiple radii
- see point of interest details

## Entities

1. PointOfInterest
2. PointOfInterestDetails

## APIs

- getLocations(latitude: Double, longitude: Double): PointOfInterestPage[]
  - we might have a lot of points in the radius, use pagination (return total, current offset)
- getDetails(pointOfInterestId: String): PointOfInterestDetails
- addPointOfInterest(details: PointOfInterestDetails): Id(String)

## Basic design

![Proximity basic design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/proximity-initial.jpg)

### Add / edit a business

- client calls the dedicated business service
- the business service persists the business metadata into a DB / cache
- the business service returns a pre-signed URL for the client to upload any larger files to blob storage (images etc)
- once the upload completes (if needed), mark the point of intersest as complete

### Get business details

- client calls the business service for an ID
- business service gets the information from the cache, if present
  - otherwise, read from DB and refresh the cache
- client pulls the large objects directly from the CDN
  - we can speed things up by keeping the business information in the CDN alltogether (with a TTL)
  - the information is a good candidate, since it changes infrequently
  - it would distribute the information closer to our customers
  - cost has to be balanced, i.e. it would depend on how much it costs
  
### Find nearby businesses

- client calls the dedicated proximity service
- service queries the DB to find nearby points 
  - since our read / write ratio is high, we can use a single leader replication
  - we can have multiple read replicas, with a tradeoff for consistency (C > A)
  - normally we should be fine with having a small delay between the business being added and it being return in the queries

## Deep dives

### Geo spatial indexing

- Redis and Postgres both support it off the shelf
- we should be able to explain how an algorithm would work
- several approaches:
  - Geohash (simplest)
  - Qhadtrees
  - Uber's H3
  
### Geohash

- the world map (square projection) is recursively divided in the quarters
  - two bits are sufficient to represent all 4 different divisions
- at each step, we divide the current square into 4 and append the sufix


![Geohash](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/geohash.jpg)

- to find a more accurate area, we simply expand the number of steps needed

![Geohash length and areas](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/geohash.jpg)

- based on the center and the radius of the search, we find the corresponding squares we need to search for
  - this is an O(1) operation
  - we need to also return adjacent geohashes. Worst case scenario is for the hashes to have no share prefix (across the initial split lines)

  - if we know which area has which points of interest, we can answer the query
  - thus, we need to store a mapping between a geohash and its points of interest
  - we are interested in storing a business for a particular length range (e.g. we don't want to store it for a square of 1000 km or 10cm)

- when a business is added, we compute the list of geohashes that are relevant for it (up and down the granularity)
- we store it in our store (with the geohash as the partition key)
- we also upload the location cache

### Scaling

- determine if we need to shard the geoindexing information
  - probably not, as it scales with the amount of businesses, not of geohashes
  - we still need multiple replicas, to work around other bottlenecks (CPU, network)

### Caching

Good candidates are:

1. The business details
  - slow changing, read frequently
  - cache mapping between businessId -> businessDetails

2. The geohash index itself
  - our repeated queries are "find all businesses in this geohash index"
  - the DB stores one row for each mapping of (geohash, businessId), since it's easier to update
  - the cache can store a map of (geohash, businessId[]) for faster retrieval
  - when a new business is added, invalidate the cache for all its geohashes
  - the cache will get populated on the next query (which is a range query based on the partition key against the table)

### Improvements

- not enough points of interest in the selected radius
  - we can expand our search by finding a higher level geohash
  - remove the last bit, which would roughly expand the area by 4x
  - recurse until we're either too far, or enough points are found

### Filtering based on business criteria

- each geohash should return a relatively small amount of points of interest
- since these are in memory, it should be ok to get the list and hydrate the details for each of them (the calls can be done in parallel)


![Proximity basic design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/proximity-final.jpg)


# Part 2 - Uber

## Requirements

### Functional

Users should be able to:

- send a start and an end location and get a fare estimate
- accept an estimate

Riders should be able to:
- accept / reject a request
- get navigation instructions between A and B

### Non-fuctional

- matching should be fast (e.g. < 1min)
- one driver assigned to one ride at one time
  - strongly consistent around ride matching
  - available everywhere else
- system should be able to handle very high request peaks (e.g. 100K requests)

## Basic design

### Models

Core entities:
- Rider
- Driver
- Ride
- Location
  - we need to know nearby drivers to we can offer them rides

### Api design

- getEstimate(start, end) -> Ride
- acceptEstimate(rideId) 

Drivers:
- updateLocation(current)
- acceptRide(rideId) -> Location (for pickup)
- updateRideStatus(rideId, status) -> Location (next checkpoint)

![Uber basic design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/uber-basic.jpg)

## Deep dives

### Ensure we can handle the flow up location updates

- estimated number of drivers: 5M
- each driver would send a location update every 5s
- 1M writes per second

Additional observation: for the OLTP flow, we just care about the latest location information
- we can use an in-memory store that would map the driver and its last updated location
  - Redis is a good candidate, as we can also query for all points from a geohash based on the distance to the Rider
  - we'll probably need more than 1 write instance, so a Redis cluster has to be configured

Problems:
- as long as the diver continues to send updates, we continuously refresh the data
- when a driver stops sending updates, we need to invalidate the data point
  - either have a separate sorted set and walk through it to invalidate
  - or leave the data there, with no TTL
    - the matching service also has to check the driver status in some other data source (this poses additional problems, as they might be out of sync)

### Invalidating location data

- a regular geohash does not allow expiring data
- a sorted set however does
- we can already think of our geohash as a map of (string -> list of drivers)

We store one sorted set per geohash. If this is at a single granularity, each driver should be present in "one" of them.
- the set name is the geohashId (for the fixed granularity)
- the userId is the key
- the timeStamp is the sorted key

- we can regularly expire keys that are older than a particular score (since it's what we sort on)

If we want to consider the map at different granularities:
- each driver would be present in multiple geohashes, each with its own granularity
  - we are probably talking about a few levels of depth, based on the spltting factor for the geohash (4 or more)

Problems:
- we now need to look at adjacent geohashes ourselves, rather than delegating this to Redis
  - we need to compute the list ourselves, but this is an O(1)
  - if our depth is constant (i.e. no expanding radiuses), we can pre-compute this and cache that list of neighbors for each of the hashes (might also work for variing radii, depends on how many levels we want)
  
### Ensure each driver is matched to a single ride

- we need to make sure a ride is only offered to a single rider
- this can be done by processing the list of riders in series (i.e. on the same thread)
  - offer to one and wait
  - if we don't hear back, offer to the next one
- we need to make sure the dirver is not offered another ride by another instance 
  - before offering the ride to a driver, acquire a short-lived distributed lock (Redis should work)
  - there are certain edge cases with this approach:
    - we acquire the lock
    - the thread gets delayed
    - the lock expires (and is acquired by another worker for another ride)
    - the initial thread resumes execution and also offers the ride
  - we might be able to live with having two push notifications, but we need to make sure the matching is done 1:1
    - when the rider accepts the ride, we need an additional conditional check that is it currently not assigned
    - ideally, we should be able to mark this in the DB somehow (fencing tokens)
    - we might want to store a "ride-accepted counter" and do a conditional update, so we'd outsource the race condition check to the DB

### Necessity of sending location updates at precise intervals

- there's an uneven value in getting the location every X seconds for all drivers
  - for someone going at high speed, we need more frequent updates
  - for a driver that's parked, there's little value in getting it every 10s
  - the client can take some inputs (e.g. speed, acceleration) and infer the frequency with which it should send updates
  - this requires a "smarter" client, which we should have (as it's an app)


### Scaling for high ride request

- the ride service does not directly call the matching service
- instead, it enqueues a message for it via an SQS queue
  - this ensures message durability
  - we return a success only after persisting to the queue
  - at this point, we're promising we'll attempt to match it (although the process hasn't started yet)
- a scaling set of workers reads from the queues and processes the items with a high degree of parallelism
  - if a worker crashes halfway through, the enqueued item will be visible again after the visibility timeout (as it wasn't marked as finished)
  - we will be processing items out of order, but that is fine, as the requests are indepent of each other

### Scaling our system further

- our service is highly partitionable based on locations (at a high level)
- we can have separate deployments handling disjoin geographical areas (e.g. at a continent / country scale)
  - we might also need to keep data local, so we do need separate deployments (e.g. for EU, China, etc.)

![Uber final design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/uber-final.jpg)

# Part 2 - Nearby friends

## Requirements

### Functional

- users should be able to see which of their friends are nearby (with a configurable radius, up to a certain range)
- users should not show up in the list if they are offline - list should be updated frequently (every few seconds)

### Non-functional

- lookups should be fast and accurate
- we should support a high volume of updates / second
- consitency over availability: we can afford to be eventually consistent
- reliability: since we're regularly updating the location, we might be ok with missing some updates
- location updates should also be persisted for analysis / ML purposes

100M DAU -> 10% online at one time 
- 10M updates every N seconds -> 1M location updates / sec
- 10M refreshes every M seconds -> 1M reads for the nearby friends / sec

## Basic design

### Models

User
- userId
- consentGiven
- friendList[]
- friendRange (configurable per user)

Location
- lat
- long

User Cache:

- userId
- Location
- TTL

### APIs

- since we're constantly sending and receiving updates, http polling would be less suited than websockets
- thus, we maintain a persistent (stateful) connection between each user and a websocket server

### Approaches

1. Keep a list of people online mapped to their corresponding geohash
- every location update for a user would write to the corresponding geohash(es)

When a request comes in:
- find the nearby geohashes, based on the user / system configuration
- intersect the list of friends with the list of people in close proximity
  - since most of the user's friends will most likely not be online and close by, it would be inefficient

- since we want the delay between "user gets nearby" and the client being aware of it, it would need to query often
- on each iteration, we'd have to perform the computation

Alternatively, we can:
-  get their list of friends
- for each friend, find their current location (if any)
- compute the distance from that location to the currenet user
- would also be wasteful, as it would result in hundreds of queries on every operation (read amplification)

2. Notify a user only when one of their friends gets nearby

- on start, the client asks for a list of online friends within the radius
- this can be a longer running operation, as it's a one off
  - we can use any of the approaches above (intersect geohash with list of friends for example)

- when a user updates their location, fire a message to all interested parties
- cache the value locally on the websocket server too
  - this would can be consumed to determine if a notification should be sent to the user, based on the distance between it and the user

**Setup**

- A is friends with B and C
- B is also friends with X and Y
- all are online except A
- B is near A / A is near B


### Initial connection flow

- A connects to a websocket server
- publish a message on channel A with the location
- update the Location Cache (with a TTL)

- get the list of friends from the Friend DB (B and C)
  - for each friend, subscribe to their pub-sub channels (B and C)

- we need to provide an initial list of close friends
  - for each friend, query the Location Cache
  - for those that have a value, check the distance from A and include if close enough (say it's just B)

### Recurring location update flow

- C sends their location to their corresponding websocket server
- the server caches the information locally
- the server pushes the new location to channel C
- the server updates the Location Cache and (optionally) updates the Historcal Location DB -> this can just be a Kafka event to be processed later (a-la Event Sourcing), or an actual write

- A's websocket server is listening on channel C
  - it receives a location update for user C
    - it needs to know which to which users it should push this information
    - what we need is a map between (channel, list of users to notify)
    - in our case, this will be (channelC, A)

  - it receives the update and checks the distance between A and C (using A's cached location)
  - if the distance is short enough, update A with C's location

The relationship between A and C is snapshotted when they log in.
- the server needs to know to which users to foward the notification
- instead, we need a separate mechanism to make sure adding / removing friends correctly reflects in the list of nearby users

### Adding / removing a friend flow

A unfriends B

- A's websocket server already has a map between (channel B, A)
- C's websocket server has a map between (channel A, B)

- the API server needs to notify it to remove A from the list in the mapping
  - since the server is listening for Cs notifications, the API server can also push a message to channel B
    - this has to contain a type, and the edge B -> A
  - we also send an inverse message, e.g. A -> B

- when the websocket server gets this message, it removes for example A from the channel B list
  - it should unsubscribe from channel B if the list is emptys
  - otherwise, we still need to listen for channel B for potentially other friends (e.g. X / Y) 

![Nearby friends design](https://raw.githubusercontent.com/strosu/learning-notes/master/books/images_system_design_interview/nearby-initial.jpg)

## Scaling

### Api servers -> horizontally scalable

### Websocket servers 

- horizontally scalable
- the only stateful componenent
  - when one disconnects, the load will immediatelly increase on all the others
  - connections should be drained before simply removing the node

### User DB

- use dynamo, let it scale to the needed levels

### Location cache

- single Redis instance is probably not sufficient
- use a Redis cluster, partition by userId -> should lead to an even distribution

### Redis pub-sub

- we need multiple instances to be able to handle a large number of channel
- channels are independent of each other, so they can be evenly distributed across instances
- we might need some orchestration on top of this, e.g. Zookeeper + consistent hashing
  - each websocket server can cache a copy of the ring locally
  - we also subscribe to updates
  - when the server wants to publish to channel X, it consults the ring to figure out which Redis instance has it
  - a similar approach is taken when subscribing

When a redis pub-sub instance is added / removed:
- no message data to be moved, as it's not persisted
- some of the channels have to also move (i.e. the map between the channel and the subscribers)

Flow:
- redis instance leaves / joins
- zookeeper notifies all subscribers
- each subscriber sends an unsubscribe message to the old server (if it's still running)
- they also send a subscribe message to the new one
- this has to be done for each channel that is moved, so it will lead to a burst of requests

- during failover, we will inevitably lose messages
- since location is something that's frequently updated, this should be an ok tradeoff

**Note** At this scale, we could have an abstraction layer over the Redis pub-sub, that should hide the fact that we are dealing with a cluster
