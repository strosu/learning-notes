# Seven Databases in Seven Weeks

Contents
========

 * [Intro](#intro)
 * [PostreSQL](#postgres)

# Intro

- The goal is to get a better understanding of the types of tools available for data storage and manipulation. 
- Any database can be used to store any data, just like you can use a screwdriver to hammer a nail. Just because you can use them does not make them the right tools for the job
- By understanding what each type of database is best at, it allows us to build systems that are simpler and better aligned with the current requirements

For each of the databases presented, the book explores:
- What type of database is it? Relational, key-value, columnar, document-oriented or graph? Each category is generally better suited for a particular pattern of work
- Why was it created? The conditions which led to its creation need to be viewed as part of a larger context, in order to understand the problems it was meant to solve
- What makes it unique? Querying on arbitrary fields, indexing, ridigity of its schema, etc
- How does it perform? Does it support replication? Sharding? Is the data kept together, or is it distributed using consistent hashing? Is the database optimized for reads or writes? 
- How does it scale? More geared towards horizontal scaling (MongoDB, HBase, DynamoDB), or vertical (Postgres, Neo4J, Redis), or something in between? 


## Database types

### Relational

- two dimensional tables with rows and collumns
- interacton is done via SQL queries
- data values are typed and may be: numeric, strings, dates, blobs or others. These types are enforced by the system via the table schemas
- tables can be morphed into new ones via joining
- covered exampls is PostreSQL (TODO - link here)

### Key Value

- simples model covered
- stores data similarly to a Dictionary
- some implementation allow defining more compelex types, or iterating through all the values, but not necessarily
- due to its simplicity, the performance can be great, but this won't be hepful with complex queries or aggregations
- covered examples: Redis, DynamoDB (TODO - link here)

Redis:
- supports complex data types like hashes and sorted sets (this is relevant for some scenarios, e.g. a learderboard can be stored out of the box).
- supports a publish-subscribe pattern, as well as blocking queues
- caches writes in memory before pushing to disk; very performant, but can lead to data loss
- great as a message broker and for holding noncritical information (if it's lost, it can be reconstructued or discarded safely). E.g. session data, worst case scenario some users need to log in again

### Columnar

- values in the same column are stored together, helping with data locallity for certain queries
- adding columns is inexpensive and is done on a row by row basis
- each row can have a different set of columns and null values are not stored, so sparse databases don't take up so much space

### Document

- stores documents: objects with a unque ID, and more fields, which can also be documents
- supports nested structures


### Graph 

- suited for representing relations between highly connected data points
- database consists of nodes and the edges between them
- easy to traverse through nodes by following the relations


# Postgres

- easy to store data and figure out what to do with it later