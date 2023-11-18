---
layout: post
title: "Introduction to Distributed Computing and Consistency Models"
author: "Krithik Vaidya"
tags: [distributed-systems, consistency]
image: /Distributed-Systems/server-rack.jpg
---

## Introduction and the need for Distribution

In recent decades, there have been exponential increases in the computing power available on a single system. However, the computational demands of certain scenarios have increased way beyond what a single system can offer. Examples being scenarios such as a web server that would need to handle tens of millions of requests per second -- there is no way that such throughput would be achievable on a single node webserver. Another example could be the case of a video-sharing website like Youtube, that needs to store petabytes of user uploaded videos -- again, no single computer can have enough storage for this.

### Defining a "Distributed System"

There are a number of different definitions of a "Distributed System", given by different subject experts. A consolidated description could be as follows: 

_A collection of computing systems which appear to behave as one, that are interconnected by a network enabling them to share messages and work towards achieving a common goal._

[Peter Alvaro](https://dl.acm.org/profile/81453654530)'s definition has a slightly different take. He says:
_A distributed system is a collection of computing nodes connected by a network and characterised by partial failure and unbounded latency_

He specifies "partial failures" and "unbounded latency" as follows:

- **Partial Failure**: A node or a few nodes of the distributed system may fail, but this should ideally not affect the overall functioning of the system.

- **Unbounded Latency**: The time that a message takes to travel from one system to another is unpredictable and has no fixed upper bound. I.e., there is no way to ensure that every message always reaches its destination within a fixed interval of time.

### Requirements of Distributed Systems

- Provide a meaningful increase in processing power, over a single node system, in Distributed Computing scenarios.
- Faster average response times and better performance under high loads, such as in distributed web servers.
- ACID compliance in distributed databases.
- Fault Tolerence, i.e. tolerant of partial failures, including failures such as network partitions, hardware failures, software crashes, etc.
- Scalability - it should ideally be easy to add/remove nodes from the system to handle more data/users, on demand.

## Clocks in Distributed Systems.

The clocks we are accustomed to - wall clocks, wristwatches, the computer's clock that shows the time, are called Physical clocks. Their purpose is to show the physical time. However, they are unsuitable for comparing the timings of different events in distributed systems, since it is impossible to keep physical clocks of different systems in sync all the time, due to factors such as [clock drift](https://en.wikipedia.org/wiki/Clock_drift), [clock skew](https://en.wikipedia.org/wiki/Clock_skew), [clock jitter](https://en.wikipedia.org/wiki/Jitter), etc. If the clocks of two communicating systems are not in sync, it will be hard to trace the causality of messages sent, making it harder to debug. The figure below illustrates this: 

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Distributed-Systems/1.jpg" alt=""></figure>

We see that in case the sequence of events at different times need to be traced in the future during debugging, Alice will claim that the value of 'A' at 10:51:48 was '10', based on the acknowledgement she received from Bob. But according to Bob the value of 'A' at the same time was '5', since at that point in time his clock was ahead of Alice's, and the write event hadn't yet occured. Hence, it is difficult to provide a consistent ordering of events. Also, clocks may jump due to miscellaneous factors such as daylight savings time, leap seconds, etc. So, in distributed systems, there is no reliable global clock.

How do we deal with this? We can exploit the fact that we do not care about the exact time at which different events occured - we are only interested in the order that they occur in.

### Logical Clocks and Causal Consistency

We will explain achieving causal consistency in case of broadcast-only messages with the example below. The system is a distributed key-value store, in which the user can ask any of the replicas to perform an operation, and that replica will broadcast the operation to the other replicas (we are considering write operations here):

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Distributed-Systems/2.jpg" alt=""></figure>

Assume that the first request sent by the user (on replica 0) reaches replica 2 soon enough but is delayed before reaching replica 1, due to some kind of network partition or delay. Replica 2 accepts this write operation. Then, a user communicates with replica 2 to invoke an update operation. Now, if this update operation reaches replica 1 before the initial message sent by replica 0 does (like in the situation shown above), replica 1 will throw an error saying that the value does not exist. Eventually, replica 1 will receive replica 0's message and accept it. But by this point the 3 replicas have diverged.

To maintain causal consistency in this scenario (i.e. writes that are related by the [happened-before](https://en.wikipedia.org/wiki/Happened-before) relation must be seen by all the processes in the same order), we need to use Vector Clocks. Each node maintains its own vector clock. If there are n nodes, each vector clock will be initialized as [0, 0, 0, ...., 0] (n zeros in total). 

When a node wishes to send a message, it will increment its corresponding position in its local vector clock, and send a copy of its vector clock along with the message. For example if there are 3 nodes in the system (node0, node1 and node2), node1 currently has vector clock [5, 4, 8] and wishes to send a message, it must increment its part of the clock to obtain [5, 5, 8].

There are also two important terms to understand:

1. Message Recieve - when a message from a sender node reaches the local buffer of the destination node.
2. Message Delivery - when the destination node actually accepts the message, removes it from the buffer and processes it.

To maintain causal consistency, a node needs to ensure the following before **delivering** a message:

_Atmost one field in the vector clock of the received message is greater than the corresponding field of the vector clock of the node, and they differ by atmost 1._

Since a node may not be able to immediately deliver every message it receives (to maintain causal consistency), it should maintain a local buffer of messages, for the messages whose immediate delivery is not possible. Whenever it delivers a message, it should inspect the buffer to see if any other messages are ready for delivery (based on the above rule) and deliver them too.

Now, consider the scenario below where vector clocks have been used to achieve causal broadcast:

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Distributed-Systems/3.jpg" alt=""></figure>

We see that replica 1 buffers the message it receives from replica 2 after it compares its local vector clock ([0, 0, 0]) with the clock in the message ([1, 0, 1]). When it receives the message from replica 0, it updates its vector clock to [1, 0, 0] and checks its buffer for any deliverable messages. It sees that the message from replica 2 is now ready for delivery, and delivers it. Hence, we see that causal consistency is maintained here. 

My implementation of a CLI causal broadcast simulator can be found [here](https://github.com/krithikvaidya/distsys-implementations/tree/main/causal_broadcast).

## The CAP theorem

The CAP theorem is a fundamental idea in Distributed Systems. It states that in a distributed system implementing a read/write storage system where all communications happen over an asynchronous network (i.e. has unbounded latency), all the following 3 properties cannot be satisfied at the same time:

**Consistency (C)** - every read sees the most recently written value. Or in other words, as mentioned [here](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html) - If operation B started after operation A successfully completed, then operation B must see the the system in the same state as it was on completion of operation A, or a newer state.
**Availibility (A)** - every request recieves a response, with the guarantee that the most recent write has occured before this request. It also implies that _any_ non failed node should be able to respond to such a request.  
**Partition Tolerance (P)** - the system can continue functioning normally even when a network partition appears between some nodes.  

In the real world, it's nearly impossible to prevent network partitions. Hence, the choice is usually between choosing linearizable consistency and losing availability when some nodes in the system go down, or remain always available and provide a much weaker consistency guarantee. There are different use cases for both -- for example, Amazon's DynamoDB prioritizies Availibility, trading it for a weaker form of Consistency. It essentially allows users to keep editing items in their shopping cart (i.e. availibility), even if the user interacts with different nodes in the system while doing this, and there is a network partition during the operation. To learn about this in depth, please refer to the excellent [DynamoDB paper](https://github.com/papers-we-love/papers-we-love/blob/master/datastores/dynamo-amazons-highly-available-key-value-store.pdf).

An example where Consistency is prioritized over availibility is MongoDB. In MongoDB, each replica set consists of a single master node, and all the other nodes are secondary nodes that replicate the master nodes data. All write requests pass through the master node. When the master node goes down (i.e. when it is partitioned away from the other nodes or crashes), a leader election algorithm takes place and the node with the most up-to-date log is elected the leader. The system remains unavailable to outside users until the new leader is elected and the secondary nodes synchronize with it. Hence, here we see that consistency is prioritized over availability.

## A Brief Overview of Consistency Models

Consistency models specify rules for operations being performed on memory. They help us understand and reason about the guarantees that distributed systems make. Some of the important consistency models are:-

**Read your writes** - A basic consistency guarantee, ensuring that a write performed by a process will be available to that process for all future reads.

**Monotonic Reads** - Ensures that when a process reads a particular piece of data, then the value of that piece of data for future reads will either be the same or a more recent value.

**Monotonic Writes** - Ensures that when a process performs a write operation on a given piece of data, that operation is completed before successive write operations operate on the same piece of data.

**Causal** - It basically incorporates Read your writes, Monotonic Reads and Monotonic Writes with some additional constraints. It ensures that causally related operations (operations that are related by the [happened-before](https://en.wikipedia.org/wiki/Happened-before) relation) appear to take place in the same order on all processes.

**Sequential** - Lamport defined sequential consistency as being met when "the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.". It can also be thought of as a stronger form of causal consistency

**Linearizable** - This deals with a single particular operation being performed on a particular object. It essentially says that once a write completes, all future reads should return the value of that write (or a more recent write). It essentially incorporates Sequential consistency with stronger constraints.

**Serializable** - This term is generally used in the context of Databases. it deals with multiple operations being performed on multiple different objects. It states that each transaction in a set of transactions (each transactions consists of some operations) is performed atomically -- i.e., there is no interleaving of operations of different transactions. I.e., it is similar to as if the transactions were executed in some serial (one-by-one) order.

**Strict Serializable** - One of the strongest forms of consistency. This basically combines Linearizability and Serializability. It essentially says that given a set of transactions, the transactions execute atomically in a serial order, and the order of these transactions corresponds to the real-time order in which they were invoked.

**Strong** - A general term, implies that only one consistent state appears among all the nodes in the system, always.

**Weak** - A blanket term implying that different nodes may observe different state until they are synchronized. Consistency models below sequential consistency fall under this.

**Eventual** - Imples that the state of all replicas eventually converge in the absence of updates.

## Exploring further:

[UCSC CSE138 Course](http://composition.al/CSE138-2020-03/)  
[MIT 6.824](https://www.youtube.com/channel/UC_7WrbZTCODu1o_kfUMq88g)  
  
[Notes on Distributed Systems for Young Bloods](https://www.somethingsimilar.com/2013/01/14/notes-on-distributed-systems-for-young-bloods/)  
[The Paper Trail Blog](https://www.the-paper-trail.org/)  
  
Textbooks -   
Designing Data-Intensive Applications, by Martin Kleppman  
Distributed Systems: Principles and Paradigms, by Andrew Tanenbaum  

## References

[https://www.youtube.com/playlist?list=PLNPUF5QyWU8O0Wd8QDh9KaM1ggsxspJ31](https://www.youtube.com/playlist?list=PLNPUF5QyWU8O0Wd8QDh9KaM1ggsxspJ31)  
[https://jepsen.io/consistency](https://jepsen.io/consistency)  
[http://www.bailis.org/blog/linearizability-versus-serializability/](http://www.bailis.org/blog/linearizability-versus-serializability/)  
[https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html)  
Distributed Systems: Principles and Paradigms, by Andrew Tanenbaum  
