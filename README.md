# git-bug data format specification

This directory specifies the data format git-bug uses to store entities (bugs,
pull-requests, …) and identities inside a git repository.


## Documents

| Document                           | Scope                                                                                                |
|------------------------------------|------------------------------------------------------------------------------------------------------|
| [Identifiers](ids.md)              | ID format and derivation, prefixes, combined IDs                                                     |
| [DAG entity format](dag-entity.md) | Base layer for all entities: commits, tree layout, operation packs, clocks, ordering, merge, signing |
| [Identity format](identity.md)     | User identities: linear version chain, public keys, fast-forward merge                               |
| [Bug entity](bug.md)               | The `bug` entity: operation types and snapshot semantics                                             |

Each entity type gets its own spec, following the structure of `bug.md`.

## Third-party implementations and extensions

Third-party clients, tools, and extensions that implement or build on this format are
welcome. The additive extensibility described below means that new operation types and
entity namespaces can be introduced independently, without modifying git-bug itself.

That said, **coordination helps avoid conflicts**. Because operation type integers are
scoped per entity and entity namespaces are plain strings, two independent parties could
accidentally pick the same value for different purposes. If you are building something
intended for broad use, opening a discussion on the git-bug issue tracker is encouraged so
that identifiers can be reserved and extensions can be made visible to the wider ecosystem.


## Design principles

### Operation-based CRDT

git-bug entities are **operation-based CRDTs** (CmRDTs). Rather than storing the current
state of an entity directly, git-bug stores the sequence of *operations* that produced that
state. The current state (the *snapshot*) is always derived by replaying those operations;
it is never stored persistently.

This design makes distributed merging conflict-free. When two replicas diverge — say, two
developers each add a comment to the same bug while offline — merging them is simply a
matter of taking the union of both operation sets. No human intervention is needed. A merge
commit with an empty operation pack records that the union has happened in git's history.

Convergence is guaranteed by replaying operations in a **deterministic total order** defined
by Lamport logical clocks: any two nodes that have received the same set of operations will
produce byte-identical snapshots. Lamport clocks capture causality; where two operations are
truly concurrent (same clock value), ties are broken by operation pack ID — arbitrary, but
stable and hard to manipulate. Each entity's operations are designed so that this ordering
produces meaningful conflict resolution (last-write-wins on scalar fields, set semantics on
collections, append-only on logs).

### Additive extensibility

The CRDT model also makes the format naturally extensible. New operation types work on new
parts of the snapshot and are ignored by clients that do not implement them. A client that
encounters an unknown operation type skips it and produces a *degraded-but-valid* snapshot:
the state it understands is correct and consistent, just incomplete from the perspective of
a fully capable client.

This means new features (a bug assignee field, a priority level, …) can be added as new
operation types without forcing existing clients to update. Entire new entity namespaces
(pull-requests, kanban boards, …) can be introduced the same way: clients that do not
implement a namespace ignore it entirely.

The same principle applies in the other direction: a client that only implements a subset of
operations can still participate in the network, read the operations it understands, and
write new operations, without corrupting the history for clients that implement more.

### Content-derived IDs

Every ID is the SHA-256 of bytes stored in git, so IDs are globally unique without
coordination and verifiable by any reader. Elements within an entity (such as comments) are
addressed by *combined IDs*. See [ids.md](ids.md).

### Format versioning

Every commit's tree carries the entity's format version. A reader checks it before decoding.
Incompatible changes bump the version and require migrating existing repositories with
[git-bug-migration](https://github.com/git-bug/git-bug-migration).

### Format stability

The format has been stable in practice because breaking changes are costly — migrating
existing repositories requires dedicated tooling (see
[git-bug-migration](https://github.com/git-bug/git-bug-migration)). No formal stability
promise is made, but the format version number encoded in every commit's tree is the
mechanism for signalling incompatible changes, and a reader must check it before decoding.

