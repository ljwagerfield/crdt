# Conflict-free Replicated Data Types

CRDTs are data structures that can be updated concurrently, without any locking or coordination, and will remain consistent.

Since a CRDT will never hold an inconsistent state, operations against a CRDT will always be correct, and will never need to be undone (e.g. through "compensating transactions"). Because of this, CRDTs are said to provide "strong eventual consistency", meaning they exhibit not only “liveness” (which states “the right thing will eventually happen”) but also “safety” (which states “a bad thing will never happen”). By contrast, regular eventual consistency only exhibits liveness.

CRDTs can be implemented as either state-based (Cv) or operation-based (Cm) [(Shapiro et al, 2011)][shapiro].

CmRDTs are operations serialized as objects (e.g. `+7`, `-2`, etc.). They must be commutative (can be played out-of-order to produce the same result) but there's no requirement to be idempotent. As such CmRDTs require your system architecture to include a message bus that ensures exactly-once delivery to all nodes (although ordering is not required).

CvRDTs by contrast represent evaluated state (e.g. `5` instead of `+7, -2`). They don't have any special requirements on the system they're used in, and as such have become increasingly popular in decentralised system architectures.

This post mainly focuses on CvRDTs.

## CvRDT (Convergent) aka 'state-based objects'

> State-based mechanisms (CvRDTs) are simple to reason about, since all necessary information is captured by the state... However, sending state may be inefficient for large objects. [(Shapiro et al, 2011)][shapiro]

### What is a CvRDT?

A single CvRDT object represents an immutable revision of a potentially distributed mutable object. A set of CvRDT objects can be ordered into a *join-semilattice* to represent the causal order of those revisions. 

For this to work, the CvRDT must be designed such that:

1.  Updates to the CvRDT are monotonic: new values must always appear greater than the value they were based off, or always less than it, if different from the original at all.

2.  Conflicting updates must produce new values which are 'siblings' to one-another (that is, both new values are 'greater than' the original value, but neither is greater than the other). We define 'conflicting updates' as being any two updates where we want both to have some observable effect on the final merged result -- i.e. we don't want one of the updates to be subsumed by the other.

3.  A resolution must always exist that allows any number of siblings to be merged into a new 'resolution' value, where that value is greater than each of those siblings. This is equivalent to saying that a monotonic update must exist for all siblings that can produce the same common value.

Given these 3 constraints, a CvRDT can be designed that allows uncoordinated updates to some distributed state, where the distributed pieces of that state can be automatically merged at any time to produce a single consistent object, without any conflicts.

### What is a "join-semilattice"?

A join-semilattice can be thought of as a DAG of sets, where each node is a union of the nodes that point into it. In a join-semilattice, there is always a single node that all nodes eventually converge to, and this node is therefore the union of all nodes in the DAG. This node is called the *'least upper bound'* (LUB).

A meet-semilattice is the same, except the edges flow in the opposite direction (i.e. a node is a _subset_ of each of the nodes that point toward it), and the single node that all nodes converge to is a subset of all the nodes in the DAG. This node is called the *'greatest lower bound'* (GLB).

A complete-lattice is the combination of the two: it's a DAG of sets that exhibits both an LUB and a GLB.

Centralised version control systems like TFS are examples of a complete-lattice, as they support merging (so always converge to a LUB) and also originate from a single shared state (a GLB). For systems which don't impose the last constraint there's only the LUB, hence we have a join-semilattice. Such systems can only grant two-way merging, since a shared ancestor is not guaranteed.

CvRDTs represent a join-semilattice: two CvRDT objects can be either equal, have hierarchy (one is a subset of the other) or are pairs (they are neither equal nor a subset of the other). Pairs must have a least-upper-bound (LUB), meaning it must be possible to automatically create a new object that is a superset (i.e. a descendent) of the two pairs. Pairs represent a branch/divergence/conflict in the data structure, and the gaurantee of the CvRDT being a join-semilattice means an automatic/natural resolution exists for all conflicts, making the datatype conflict-free.

![CvRDTs produce a monotonic join-semilattice][semilattice]

Monotonicity ensures that objects resulting from non-concurrent updates can be ordered in the sequence they occurred. Assuming a non-decreasing data type, any lower-valued object can be treated as past information or a subset of the current information, allowing it to be discarded.

### Using CvRDTs to detect conflicts in regular data types

Two-way merge detection of any data type can be achieved by attaching a CvRDT as a header to the underlying payload. Note that such headers can only provide *detection* and *commitment* to the merged result; developers must still decide whether merging options should be deferred to the user (like in source control), or if an auto-merge is possible through designing operations in a way that makes them idempotent and commutative.

Regardless, any CvRDT that supports unique increments can be used for the header, since **all CvRDTs will produce the same shape graph given the same set of concurrent unique increments.** That said, the vector clock (aka `vclock`) is currently one of the most efficient CvRDTs considering its payload and associated compare/merge functions. Consequently it is used in many distributed systems, [including Riak][riak].

*As an example, a grow-only set (aka `g-set`) could be used whereby each update inserts a GUID. This would produce a monotonic join-semilattice, but would be far less efficient than a `vclock`.*

### Vector clocks

Vector clocks form a monotonic join-semilattice; all pairs have a LUB - a descendant value which they both converge to and is idempotent, since the LUB is not a pair with either of its inputs:

    initial:  A1:B1                             // nodes b and c start with this value
    node b:   A1:B1 + B1        =  A1:B2        // inc
    node c:   A1:B1 + C1        =  A1:B1:C1     // inc
    node d:   A1:B2 + A1:B1:C1  =  A1:B2:C1:D1  // merge
    
    idempotent:
    A1:B2:C1:D1 + A1:B2 = A1:B2:C1:D1

Vector clock with user-defined payload:

    A1:B2:C1:[X] + A1:B2:D1:[Y] = A1:B2:C1:D1:[MERGE(X,Y)]
    
#### Vector clock redundancy with CvRDT payloads

Vector clock headers become redundant when used with CvRDT payloads, since the payload is capable of identifying its own pairs and producing valid LUBs. Fortunately, the pairs identified by the `vclocks` will be a superset of the pairs identified by the payload, so merging for each `vclock` pair will at worst produce redundant merge operations. This is because the `vclock` guarantees unique increments, whereas the payload may not: consider a set of weekdays observed by the application - objects identified as pairs by the `vclock` could possess duplicate payloads.

    A1:B2:C1:[{MON, TUES}] + A1:B2:D1:[{MON, TUES}] = A1:B2:C1:D1:[{MON, TUES}]

Such redundancy may arise in distributed databases which assume all payloads to be non-CvRDTs; or rather, a distributed database that hasn't implemented support for user-defined CvRDTs. Just as an example: Riak will attach `vclocks` to payloads, creating said redundancy if your payload is itself a type of CvRDT.

### Simple CvRDT

The grow-only set (aka `g-set`) is a natural CvRDT. Therefore, the simplest approach for implementing a CvRDT is to emulate an operation-based data type by storing commutative operations combined with a unique discriminator into a `g-set`. These operations can then be replayed (as described in the CmRDT section) to evaluate the result.

This approach may be inefficient when compared to more tailored algorithms (i.e. counters are better implemented as `vclocks`).

## CmRDT (Commutative) aka 'ops-based objects'

> Specifying operation-based objects (CmRDTs) can be more complex since it requires reasoning about history, but conversely they have greater expressive power. The payload can be simpler since some state is effectively offloaded...  [(Shapiro et al, 2011)][shapiro]

### What is a CmRDT?

A CmRDT payload simply expresses an operation (e.g. `-10`, `+20`, etc.).

The operation must be commutative (meaning they can be played in any order to produce the same result), but does not have to be idempotent (meaning the operation doesn't need to worry about being accidentally replayed many times).

Due to the lack of requirement for idempotency, CmRDTs require infrastructure that can gaurantee exactly-once delivery semantics of CmRDT payloads between nodes (although ordering is not required).

This is closely related to [event sourcing][eventsourcing].

[shapiro]: http://hal.upmc.fr/docs/00/55/55/88/PDF/techreport.pdf  "A comprehensive study of Convergent and Commutative Replicated Data Types, Shapiro et al (2011)"
[riak]: http://docs.basho.com/riak/latest/theory/concepts/Vector-Clocks/  "Vector Clocks in Riak"
[eventsourcing]: http://martinfowler.com/eaaDev/EventSourcing.html  "Event Sourcing by Martin Fowler"
[semilattice]: images/monotonic-join-semilattice.gif  "CvRDTs produce a monotonic join-semilattice"
