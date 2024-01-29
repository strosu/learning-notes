# Building blocks

## Data structures

### LSM trees deep dive

## Tools

### Redis Pub/Sub

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


### Redis sorted sets

Properties:

- each member is associated with a value
- members must be unique (e.g. username), but the value can repeat itself (e.g. user's score)
- the value is used to rank the members is an ascending order; two members with the same value will be ordered lexicographically 
- useful for a leaderboard, or a rate limiter
- represented by a hashset (mapping a member to a value) and a **skip-list**, which maps values to members

**Skip list**

- a linked list that stores the elements in order
    - adding and removing is an O(1) operation
    - indexing is an O(n) operation
- to improve the list traversal

### ElasticSearch

Properties:

- allows searching on documents persisted inside it
- data is sent in JSON format
- can do point(?) queries via GET
- can do range queries via the Search endpoint
    - returns the documents that match the query; can return just a subset of fields if specified in the query

## Problems
