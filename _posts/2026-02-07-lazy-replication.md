---
layout: single
title: "Replicating lazy replication in python"
date: 2026-02-07
# toc: true
# toc_sticky: true
---

I've been reading about distributed systems lately. I have a lot to catch up on.
When I started making a reading list a couple months ago, I had heard about the
CAP theorem, that it was about tradeoffs in consistency, availability, and
P-something, so I started there.

The CAP acronym stands for "Consistency of reads[^1], Availability of writes,
and presence of a network Partition." The theorem (poorly stated) says you can
pick any two, but not all three.[^2] If your service allows consistent reads (all
replicated nodes return the same value) and is always available for write
operations, then the system cannot function during a network partition. A
network partition is when not all nodes can communicate with each other. And if
you want to allow writes to proceed even when there's a network partition, then
you cannot guarantee that every node will return the same value on read.

[^1]: The "C" is sometimes phrased "Consistency of replicas" rather than consistency of reads, which is a fine distinction. Systems are only observed through read operations, so inconsistency can only be noticed through reads; however, if a node fails and its work hasn't successfully been replicated anywhere, then a future consistent read becomes impossible. So consistent replicas and consistent reads are closely tied but have different implications for how you might design the system to handle failure.
[^2]: The original proof gives a precise definition of the CAP conjecture original posed by Brewer: [https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf). Besides being more precise, it also describes weaker forms of consistency that are useful in real systems.

The CAP theorem as a _theorem_ is correct, but the impossibility result has been
used as a teaching tool and a framework for thinking about tradeoffs in a
distributed, replicated service. And as a framework it's not very useful. It
neglects the fact that users can tolerate some types of inconsistent reads, but
not others; writes can proceed during a network partition under certain
conditions, but not others; that there are many kinds of failure besides just a
network partition (slow responses, Byzantine nodes, node failures, and an actual
break in the network connection between parts of the network that otherwise
function well).

In other words, the CAP framework is simplistic. I shopped around for better
frameworks and found an excellent paper, ["Rethinking Eventual
Consistency"](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/sigtt611-bernstein.pdf)
by Philip Bernstein and Sudipto Das from Microsoft. They give a more complete
classification of the tradeoffs made in modern distributed systems and how to
think about them.

First, they acknowledge that availability and network partition tolerance are
essential for most services, so it's _read consistency_ that has to be
adjusted. (For what it's worth, the original CAP proof paper also acknowledges
this.) Then, they disambiguate types of consistency and their uses. For example,
in an email system, it's usually enough to offer _causal_ consistency where a
single user always observes their own updates and the updates they've observed
before. They don't observe the whole system at once, so the replicas don't have
to be consistent.

The whole paper is worth reading. I myself focused on just one part that I found
interesting: eventual consistency through partial ordering, implemented with
vector clocks. The main reference for this in the eventual consistency paper is
["Providing high availability using lazy
replication](https://www.cs.princeton.edu/courses/archive/spr24/cos418/papers/lazy.pdf), by Ladin et al.

In this paper, a service is replicated across a network of symmetric nodes that
each serve both reads and writes to clients. The client uses a front end
service, and this front end service has duties like routing to a particular
preferred node and coordinating with nodes about which updates it expects to
see. This makes the replication transparent to the user while storing some
important program state on the client's side.

A system is causally consistent if each user has a consistent view of the
system: they see the things they've seen before, and the system reflects the
updates they've made (maybe in response to things they've seen) in the right
order. From the perspective of an email client, the exact order of unrelated
emails being sent in the system is unimportant. But if I read an email and then
refresh my messages, I should still see that email (my view should never go
"back in time"), and the messages shouldn't be reordered. If I reply to an
email, and someone else replies to my reply, the thread should appear in the
same order for everyone, regardless of which replica they talk to (replies are
causally linked).

This results in chains of causality flying back and forth from client to replica
and between replicas (as they share updates with each other). The _model_
described in the paper uses actual chains of causality. The _implementation_ of
course is limited by memory and bandwidth, where full chains of causality are
inconvenient.

Instead, the system is implemented using _vector clocks_, which are a nifty
trick for tracking dependencies.

To understand vector clocks consider that the system is changed through updates,
and it's observed through reads. A read observes some data item, which is the
product of all of the updates that have affected that data item, performed in
some order.

The state at a particular replica is then defined by the log of updates it has
processed. As long as the replica itself keeps track of its log of updates,
everyone else can refer to a particular state of that replica by the _length of
its log_, or in other words the number of updates that replica has processed.

This is the underlying idea of a vector clock. For N replicas, the state of the
system is identified by a vector _v_ of length N, where each element
_v<sub>i</sub>_ is the number of updates processed at replica _i_.

This compact representation of the system is useful but not sufficient. The
Ladin, et al., paper goes into the details of how their system makes use of
vector clocks to enforce several types of operations (causal, forced, and
immediate, in increasing order of strictness).

# Simulating the system
I was having a hard time visualizing and playing with this system on paper, so I
wrote a python simulation of the key parts. You can find it here:
[https://github.com/charlie-gallagher/simulation-lazy-replication-ladin-1992](https://github.com/charlie-gallagher/simulation-lazy-replication-ladin-1992)

It does somewhat detailed logging of what's going on at each node (node=replica)
and client, and in the end prints out a summary of what went on. Here's an
example summary:

```
❯ python3 node.py

======================
Summary
  Nodes: 4
  Front ends: 10
  Current time: 99
  Stats: {'updates': 316}
--------------------------
FrontEnd (0)
  Preferred node: 0
  Prev: [78, 96, 50, 87]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 177
  Seen vals: [12, 7, 9, 20, 20, 29, 40, 61, 86, 89, 105, 119, 110, 108, 119, 135, 145, 147, 153, 177]
  Stats: {'updates': 30, 'update_completes': 30, 'query_starts': 20, 'query_completes': 20, 'failed_polls': 0}
FrontEnd (1)
  Preferred node: 3
  Prev: [78, 98, 52, 88]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 170
  Seen vals: [6, 9, 21, 23, 33, 62, 115, 120, 108, 117, 112, 131, 133, 147, 153, 170]
  Stats: {'updates': 34, 'update_completes': 34, 'query_starts': 16, 'query_completes': 16, 'failed_polls': 0}
FrontEnd (2)
  Preferred node: 1
  Prev: [78, 98, 50, 87]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 173
  Seen vals: [6, 18, 11, 18, 23, 61, 115, 108, 127, 133, 141, 173]
  Stats: {'updates': 38, 'update_completes': 38, 'query_starts': 12, 'query_completes': 12, 'failed_polls': 0}
FrontEnd (3)
  Preferred node: 1
  Prev: [76, 97, 42, 84]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 155
  Seen vals: [5, 2, -2, 4, 12, 8, 9, 19, 27, 33, 37, 62, 86, 96, 100, 114, 110, 112, 115, 110, 115, 127, 127, 128, 148, 141, 155]
  Stats: {'updates': 23, 'update_completes': 23, 'query_starts': 27, 'query_completes': 27, 'failed_polls': 0}
FrontEnd (4)
  Preferred node: 1
  Prev: [78, 98, 50, 87]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 173
  Seen vals: [-2, 6, 9, 18, 21, 29, 72, 96, 117, 129, 134, 137, 141, 148, 143, 153, 173]
  Stats: {'updates': 33, 'update_completes': 33, 'query_starts': 17, 'query_completes': 17, 'failed_polls': 0}
FrontEnd (5)
  Preferred node: 2
  Prev: [64, 81, 51, 70]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 131
  Seen vals: [5, 9, 8, 12, 11, 20, 38, 62, 105, 119, 114, 111, 115, 118, 131]
  Stats: {'updates': 35, 'update_completes': 35, 'query_starts': 15, 'query_completes': 15, 'failed_polls': 0}
FrontEnd (6)
  Preferred node: 1
  Prev: [74, 98, 35, 79]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 148
  Seen vals: [4, 5, 11, 9, 32, 62, 66, 76, 93, 100, 120, 111, 108, 118, 123, 128, 133, 141, 148]
  Stats: {'updates': 31, 'update_completes': 31, 'query_starts': 19, 'query_completes': 19, 'failed_polls': 0}
FrontEnd (7)
  Preferred node: 0
  Prev: [78, 93, 46, 86]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 168
  Seen vals: [-5, 9, 5, 5, 5, 18, 21, 20, 32, 41, 62, 66, 120, 112, 115, 116, 117, 135, 131, 136, 144, 168]
  Stats: {'updates': 28, 'update_completes': 28, 'query_starts': 22, 'query_completes': 22, 'failed_polls': 0}
FrontEnd (8)
  Preferred node: 3
  Prev: [74, 85, 35, 88]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 147
  Seen vals: [4, 17, 11, 9, 18, 23, 30, 37, 62, 96, 89, 105, 115, 107, 118, 131, 135, 133, 147]
  Stats: {'updates': 31, 'update_completes': 31, 'query_starts': 19, 'query_completes': 19, 'failed_polls': 0}
FrontEnd (9)
  Preferred node: 2
  Prev: [65, 82, 52, 71]
  Is blocked on query: False
  Is blocked on update: False
  Last received value: 138
  Seen vals: [5, 5, 18, 15, 8, 20, 23, 22, 38, 40, 41, 62, 68, 76, 120, 110, 138]
  Stats: {'updates': 33, 'update_completes': 33, 'query_starts': 17, 'query_completes': 17, 'failed_polls': 0}
--------------------------
Node (0)
  Log records: 0
  First 5 log records:
  Replica TS: [78, 98, 52, 88]
  Value: 170
  Value TS: [78, 98, 52, 88]
  TS Table: [[78, 98, 52, 88], [78, 98, 52, 88], [78, 98, 52, 88], [78, 98, 52, 88]]
  Gossip queue length: 0
  Update queue length: 0
  Query queue length: 0
  Query results length: 0
  Update results length: 0
  Stats: {'updates': 78, 'gossip_messages_processed': 243, 'gossip_updates_processed': 238, 'queries': 49}
Node (1)
  Log records: 0
  First 5 log records:
  Replica TS: [78, 98, 52, 88]
  Value: 170
  Value TS: [78, 98, 52, 88]
  TS Table: [[78, 98, 52, 88], [78, 98, 52, 88], [78, 98, 52, 88], [78, 98, 52, 88]]
  Gossip queue length: 0
  Update queue length: 0
  Query queue length: 0
  Query results length: 0
  Update results length: 0
  Stats: {'updates': 98, 'gossip_messages_processed': 237, 'gossip_updates_processed': 218, 'queries': 47}
Node (2)
  Log records: 0
  First 5 log records:
  Replica TS: [78, 98, 52, 88]
  Value: 170
  Value TS: [78, 98, 52, 88]
  TS Table: [[78, 98, 52, 88], [78, 98, 52, 88], [78, 98, 52, 88], [78, 98, 52, 88]]
  Gossip queue length: 0
  Update queue length: 0
  Query queue length: 0
  Query results length: 0
  Update results length: 0
  Stats: {'updates': 52, 'gossip_messages_processed': 238, 'gossip_updates_processed': 264, 'queries': 38}
Node (3)
  Log records: 0
  First 5 log records:
  Replica TS: [78, 98, 52, 88]
  Value: 170
  Value TS: [78, 98, 52, 88]
  TS Table: [[78, 98, 52, 88], [78, 98, 52, 88], [78, 98, 52, 88], [78, 98, 52, 88]]
  Gossip queue length: 0
  Update queue length: 0
  Query queue length: 0
  Query results length: 0
  Update results length: 0
  Stats: {'updates': 88, 'gossip_messages_processed': 284, 'gossip_updates_processed': 228, 'queries': 50}
======================
```

It successfully replicates updates[^3] and produces the same value at each
replica, in this case `Value: 170`.

The system isn't perfect, and it still probably has some bugs, which have been
difficult to track. But the value was in the exercise, and if you're interested
in the Ladin paper, this is a functioning example that's simple enough to be
considered basically pseudocode.



[^3]: A running total. This actually isn't sensitive to the order in which operations are run, a so-called CRDT data type (see _Rethinking Eventual Consistency_ paper). This was a good starting point, I thought, because I didn't have to work out the partial ordering just yet.

