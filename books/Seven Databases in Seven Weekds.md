# Seven Databases in Seven Weeks

Contents
========

 * [Intro](#intro)
 * [Assumptions](#assumptions)
 * [Structure](#structure)
 * [Observations](#observations)
 * [General approach](#general-approach)
 * [Abstractions](#abstractions)
 * [Next Steps](#next-steps)

### Intro

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