Conflict-free Replicated Data Types
====

CRDTs offer 'Strong Eventual Consistency': a flavor of eventual consistency that ensures conflicts can be merged automatically to produce a value that is guaranteed to be correct/consistent. CRDTs can be implemented as both state-based (Cv) and operation-based (Cm) [(Shapiro et al, 2011)][shapiro], although developers will typically opt for the most suitable route given their requirements. CvRDT is arguably a more complex subject and is hence the main topic of this article.

## CvRDT (Convergent) aka 'state-based objects'

> State-based mechanisms (CvRDTs) are simple to reason about, since all necessary information is captured by the state... However, sending state may be inefficient for large objects. [(Shapiro et al, 2011)][shapiro]

### What is a CvRDT?

CvRDTs are objects which can be ordered into a *join-semilattice*, where causal ordering is guaranteed by ensuring objects are updated *monotonically* and concurrent non-idempotent writes produce a branch.

A join-semilattice can be thought of as an inverted rooted tree; a tree whereby any node may have multiple parents, but ultimately converge to a single leaf or *'least upper bound'* (LUB for short). This contrasts a meet-semilattice, which can be thought of as a regular rooted tree, where the root is the *greatest lower bound*.

The join-semilattice represents a version graph where ancestors can diverge, hence making three-way merges impossible. This occurs when objects are initialized or updated by disconnected nodes; such writes form branches in the join-semilattice.

Two objects can be either equal, have hierarchy (one descends the other) or are pairs; the latter signifies a branch/divergence/conflict. There must be enough intrinsic state within the two objects to determine this.

Pairs must have a least-upper-bound (LUB): a new descendant object whose parents are the two merged objects. This is a constraint of the join-semilattice, and hence ensures a single convergent leaf.

![CvRDTs produce a monotonic join-semilattice][semilattice]

Monotonicity ensures that objects resulting from non-concurrent updates can be ordered in the sequence they occurred. Assuming a non-decreasing data type, any lower object can be treated as past information or a subset of the current information, and can hence be discarded as the current state already contains all its information.

### Using CvRDTs to detect conflicts in regular data types

Two-way merge detection of any data type can be achieved by attaching a CvRDT as a header to the underling payload. Any CvRDT that supports unique increments can be used since **all CvRDTs will produce the same graph given the same set of concurrent unique increments.**

That said, the vector clock (aka `vclock`) is currently the most efficient data type considering its payload and associated compare/merge functions. Consequently it is used in many distributed systems, [including Riak][riak].

*As an example, a grow-only set (aka `g-set`) could be implemented whereby each update inserts a GUID. This would produce a monotonic join-semilattice, but would be far less efficient than a `vclock`.*

### Vector clocks

Vector clocks form a monotonic join-semilattice; all pairs have a LUB - a descendant value which they both converge to and is idempotent, since the LUB is not a pair with either of its inputs:

    A1:B2:C1 + A1:B2:D1 = A1:B2:C1:D1
    A1:B2:C1:D1 + A1:B2:C1 = A1:B2:C1:D1
    A1:B2:C1:D1 + A1:B2:D1 = A1:B2:C1:D1

Vector clock with user-defined payload:

    A1:B2:C1:[X] + A1:B2:D1:[Y] = A1:B2:C1:D1:[MERGE(X,Y)]
    
#### Vector clock redundancy with CvRDT payloads

Vector clock headers become redundant when used with CvRDT payloads, since the payload is capable of identifying its own pairs and producing valid LUBs. Fortunately, the pairs identified by the `vclocks` will be a superset of the pairs identified by the payload, so merging for each `vclock` pair will ensure necessary merges for the payload are not missed. This is because `vclocks` guarantees unique increments, whereas the payload may not: consider a set of weekdays observed by the application - objects identified as pairs by the `vclock` will actually possess convergent payloads.

    A1:B2:C1:[X] + A1:B2:D1:[Y] = A1:B2:C1:D1:[LUB(X,Y)]

Such redundancy may arrise in systems which assume all payloads to be non-CvRDTs; or rather, a system that hasn't implemented support for user-defined CvRDTs. One such example is Riak.

### Simple CvRDT

The grow-only set (aka `g-set`) is a natural CvRDT. Therefore, the simplest approach for implementing a CvRDT is to emulate an operation-based data type by storing deltas/operations with a unique discriminator as a `g-set`. These deltas can then be replayed (as described in the CmRDT section) to evaluate the result.

This approach may be inefficient when compared to more tailored algorithms (i.e. counters are better implemented as `vclocks`).

## CmRDT (Commutative) aka 'ops-based objects'

> Specifying operation-based objects (CmRDTs) can be more complex since it requires reasoning about history, but conversely they have greater expressive power. The payload can be simpler since some state is effectively offloaded...  [(Shapiro et al, 2011)][shapiro]

### What is a CmRDT?

Operations are appended to an external shared event log / message queue. Operations can then be replayed downstream by any replica to reach an eventually consistent value. The object payload contains a snapshot of the most recently calculated value.

This is the essence of [event sourcing][eventsourcing].

[shapiro]: http://hal.upmc.fr/docs/00/55/55/88/PDF/techreport.pdf  "A comprehensive study of Convergent and Commutative Replicated Data Types, Shapiro et al (2011)"
[riak]: http://docs.basho.com/riak/latest/theory/concepts/Vector-Clocks/  "Vector Clocks in Riak"
[eventsourcing]: http://martinfowler.com/eaaDev/EventSourcing.html  "Event Sourcing by Martin Fowler"
[semilattice]: images/monotonic-join-semilattice.gif  "CvRDTs produce a monotonic join-semilattice"
