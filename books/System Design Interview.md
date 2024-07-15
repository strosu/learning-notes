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