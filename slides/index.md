***
##Modern Architecture Patterns

***
### A Story

' [3 min]
'
' [John]
'
' Once upon a time, there was a small company run by a single person. He used to take down his orders on a notebook with a pen, and tick the orders off as he was done filling them. At the end of the week, he would tally up the totals, and go have a beer and celebrate. And then he would start the next week. He would know everything about the company the instant it happened - he *was* the company after all.
'
' Then he started getting more orders than he could handle himself. So someone he knew built a small application on Microsoft Access. He used to run it on the "company computer", and now he sat at a desk while some youngsters did all the running around. He still knew everything about his company's activities, but only after his workers told him - and they often didn't know the details of what their colleagues were up to, but the owner could put the pieces together after talking to all of them.
'
' Now the company grows some more. This little Microsoft Access app cannot be used any more because multiple people are creating orders and fulfilling them, and the boss doesn't want to be the bottle neck entering everything in the "company computer". So someone tells the boss about this "internet" thing, so they knock up a website in PHP and runs it on MySQL, buy a server and runs it from the closet. Someone prints out a report and gives it to the boss once a week, with charts and summaries of the previous week's activities.
'
' Now the company gets some branches in different cities...then different countries...and different time zones. They start selling on Amazon and eBay. They've moved to a data center, with a large Oracle database and big servers. Now there are managers that get the weekly reports, and put together a monthly statement for the boss. The boss only gets a big picture view of where the business is doing well and where it needs to improve, and how they need to plan for Chinese New Year in China and Black Friday in the US and Christmas all over the place. 
'
' Now they're running enterprise components, and they have issues with scale and reliability. Every time they have a slow down, the full-time DBA says something about transactions and locking, and the full-time System Administrator talks about putting an upgrade of some sort in there, and raises a purchase order for another 10k.
'
' Then one day, it crashes hard, and they have to rebuild the whole server - it's going to cost them 3 months of time to get the system back up from buying the server and disks to installing and testing the web application again - and then someone tells them about the cloud. They're not sure if that's actually going to give them the best kind of solution - after all, the heart of the system is still slow - and now they ask us what they need to do to make things better...

***
###Modern Architecture Patterns

' [1 min]
'
' [Mahesh]
'
' Welcome to Modern Architecture Patterns. I'm Mahesh Krishnan and my co-presenter is John Azariah. We come here to sunny Oslo from Melbourne, Australia. I'm the Head of Products and Client Services with Readify, and John is a Software Architect with Applied Data Science. We're both Azure MVP's, and we're glad you came to our talk on Modern Architecture Patterns for the Cloud.
'
' A great many people we speak to have a similar story to the one you just heard, and we have had to help them design applications to take advantage of the cloud. As we hope you will discover from this talk, the cloud is an exciting platform which allows us to take some creative approaches to build solutions!

* @maheshkrishnan 
* @johnazariah

***
### This 'Cloud' Thing

' [3 min]
'
' [Mahesh] gives the introduction to the cloud architecture. 

* COTS Hardware
* Elasticity on demand
* Zero capital outlay - Pay as you go
* Scales to virtual infinity

***
### n-Tier Applications

' [1 min]
'
' [Mahesh]
'
' Let's take a good look at how the web application would have been built in our story
' We would've had a single central database, a little server farm with stateless middle-tier components, a load-balancer, and many remote clients to individual users.
' You couldn't risk the system going down, so the hardware was enterprise-grade - dual power, RAIDed disks - that sort of thing.
' We would've built these servers to industry specifications - backups are carefully taken often, and upgrades to the server components are performed with a great deal of care and prayer, and everyone was on call to make sure stuff got fixed if something broke.
' When we wanted more capacity, we would add more middle-tier servers, beef up the load-balancer to ensure that a single middle-tier server didn't overload. The database could be mirrored and there was a hot-standby in case the unthinkable happened and the primary server went down. But mostly, we just put more memory and faster disk on those to increase capacity.

* Central Database
* Stateless Middle-Tier
* Remote Client

*** 
### Distributed Systems
' = TBD - have to split this out into individual slides
'
' Now there's a reason why we increased capacity in our database by throwing more tin at the problem instead of adding new servers like we did in the middle-tier. The middle-tier is stateless, so increasing capacity by scaling out is easy, but when there is state involved, things get a little more tricky.
'
' image - big denormalized table
'
' Now most traditional databases effectively look like this - we may normalize things out differently, but effectively, we have the state of the system laid out this way. The great thing about this system is that whenever something changes in the state of the system, the state of the whole system changes in a way that preserves consistency - that is, we are able to make a change, run a query, and see the change synchronously.
'
' So what happens if we try to scale things out? Let's say we have two copies to serve data simultaneously.
'
' image - two big normalized table
'
' Let's say that a change gets applied to one side. In order for us to preserve linearization consistency, we would have to apply the same change *synchronously* on the other side, so that a query on the other side reflects the change immediately. And vice versa. This can be done, but it effectively requires a technique known as "two phase commit" - which only accepts a change on one side if the other side acknowledges that it can also accept it.
'
' Now think about what happens when there is a network partition - and the two servers cannot speak to each other. In that case, two-phase commit will, very correctly, prevent updates from being taken - reads will still work on either side - reflecting the last change that happened before the partition - but writes will not be accepted at all. This is why two-phase commit is not "partition-tolerant".
'
' Such systems provide strong consistency - linearizability - at the expense of partition-tolerance and partial unavailablity, and therefore do not support the scale-out model of increasing capacity.
'
' Let's consider several approaches one by one to see how we can achieve scaling out of state:
'
' image - sharding
'
' One approach which is becoming quite popular in the RDBMS community is to group certain rows together and create "shards" of the data which are effectively independent of each other. For example, customers making orders and purchases do not directly affect each other, so they could, theoretically, be stored in independent databases. This obviously allows for strong consistency within each shard, and a degree of availability as the failure of one shard does not affect availability of the others.
'
' Where the price is paid is when a report needs to be run on all customers - then there is no guarantee that all shards will respond - or even if they have reflected all the changes applied to them - so we sacrifice consistency for partition-tolerance with this model. 
'
' That said, mainstream databases like Oracle and SQL Server are beginning to support this model so you don't really need to change the way you code, but as long as you are aware of the reduced consistency guarantees, you can get a measure of scale-out on your data-set.
'
' image - column-store
'
' Another approach is to split the data-set vertically, and put all the data associated with a single column in an independent store. Typically these will be stored as key-value pairs keyed with the entity-id, and this is the basis of such systems as Google's "Big-Table". 
'
' This system works really well when each attribute (or column) of the data is modified by different systems, possibly at different frequencies. For example, the address of a customer may change very rarely compared to their account balance. In the traditional approach, we would have to acquire some form of lock on the customer to update the address - which might affect the performance of the order-management system but with this approach you don't have that problem.
'
' Obviously, to get a full picture of the customer, one would have to query across multiple partitions and synthesize a report - which may be expensive, but cross-customer queries which were hard to do in the sharded approach could be made much easier with this one.
'
' Again, we get the ability to scale out and survive partition failures - failures only affect columns independently - but there are no cross-column consistency guarantees.
'
' image - stateful-microservice
'
' A third approach may be to treat each entity-attribute as a minimal grain of state, and have a way of storing each of those independently.
'
' Such a system can be written using the Actor model, for example, and represents a micro-service capable of changing just that bit of state independently of every other bit of state. 
'
' Systems like Microsoft "Orleans" have been used to build systems for the "Halo" game where billions of objects are stored and information is exchanged with very low latencies.
'
' Typically such systems provide extremely high levels of scalability, but also are much more tricky to design. Reporting tends to be aggregation of state, and again, we achieve partition-tolerance and availability at the expense of strong consistency.
'
' image - consensus-based storage systems
'
' = TBD =
'
' image - CAP theorem
'
' This theme seems to be appearing every time we talk about distributed state - we are looking for a system that is consistently linearizable across reads and writes, fully available when parts of the system have failed, and able to survive a network partition - and we only seem to be able to get some of these characteristics.
'
' It turns out that such a system cannot be built - there's a theorem called the CAP Theorem that limits the capabilities of a distributed system to at most two of the three characteristics.
'
' As we have seen, two phase commit systems are advertised as "CA" systems, but we've seen that whilst they are linearizably Consistent, they are actually only partially Available. Strictly speaking, they are "C" systems.
'
' Other systems claim to be CP or AP, but generally speaking, if you look closely, they tend to be just "P" systems - which is great in itself, and usually sufficient for most applications.
'

***
### Eventual Consistency

***
### Command-Query Separation

***
### Event-Sourcing

* Record Events that affect state instead of modifying state
* Compute state periodically and memoize
* Consistent to a point-in-time
* Multi-Master Synchronization Scenarios
* Timing and Clock issues

***
### Map-Reduce

***
### Lambda Architecture (?)
* Handle bulk data at rest and fast-changing data in flight

***
### Queue-Centric Workflows

* Queue-Coupling provides resilience
* Queue-Coupling provides elasticity and scale
* Queue-Coupling provides 
