# Designing Data Intensive Applications

My notes on Martin Kleppmann's book, with some personal additions on the topics he explores.

Contents
========

 * [Stream processing](#stream-processing)

 # Stream processing

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

Examples where no intermediares are used:
- UDP multicast
- TCP/IP multicast
- direct API calls through REST or RPC, if the customers support it; in effect a webhook, where consumers are notified of new data

In general, the assumption is that both producer and consumers are online all the time. You can implement retries and other similar mechanisms, but the design is not very robust.

### Message brokers

- Represents an intermediary that both the producers and consumers connect to. Clients can come and go, the problem has now moved to the broker itself.
- Some store the events only in memory, while others persist them
- as opposed to direct cals, the system is now asynchronous. Usually the producer gets an success once the message has been processed by the broker, with little alternatives to be notified when the consumers get it.

Comparison with Databases:
- DBs are persistent, whereas message brokers usually store messages until they are processed
- no "traditional" querying or indexing; however, consumers are notified when there is new data

Message delivery:
- either distributes the messages between the consumers in a RR fashion, acting as a load balancer
- distributes them to ALL the consumers, so multiple systems can process the output
- a mix between the two: distribute them to ALL groups of consumers, while load balancing within the consumers in the same group

Acknowledgements and retries
- brokers wait for the consumers to ACK they received and processed the message; only then it is removed from the queue
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

