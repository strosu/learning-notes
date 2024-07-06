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