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

The basis of stream processing is an **event**. This is a small, self contained, immutable object, representing the details of something that happened, at some point in time. 

