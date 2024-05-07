# Designing Data Intensive Applications

A more in depth reading and notes from DDIA

# Chapter 1

## Reliability
- the system should continue to work correctly when dealing with advertity (to the most of its ability)
    - ties strongly into consistency, i.e. failures along durign a request's execution should not leave the system in an inconsistent state
- **fault tolerance** refers to one of the components of the system not behaving as expected. 
    - we cannot reduce the change of failures to zero (human/hardware/software errors), thus it is only a matter of time until a failure occurs. 
    - thus, any reliable system should take these into account and work on mitigating them as much as possible

### Hardware failures

- since most services are deployed in a hosted manner (AWS, Azure), there are no strong guarantees of VM availability
    - on the contrary, machines becoming unavailable is a routine occurrence that should be planned for
- the cloud providers themselves are responsible for ensuring reliabiity at this level usually, e.g. by spinning up a new VM when one dies

## Scalability
- the system should be able to accomodate growth: both in the total amount of data it stores, as well as the incoming rate. 
- we should also be able to accomodate more complex data coming in as the requirements evolve over time

## Maintainability
- the various (and changing) set of people working on the system should be able to do so *productively*, e.g. 