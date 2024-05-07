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




### ElasticSearch

Properties:

- allows searching on documents persisted inside it
- data is sent in JSON format
- can do point(?) queries via GET
- can do range queries via the Search endpoint
    - returns the documents that match the query; can return just a subset of fields if specified in the query

## Problems
