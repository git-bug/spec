# Specification: DAG entity format

This document is the formal specification for the generic data format used by all entities in
git-bug (bugs, pull-requests, etc.). It is intended for contributors and for authors of
third-party clients that read or write git-bug data.

For the motivation behind these design choices, see [Data model - the rational](../design/data-model.md).


## 1. Overview

An *entity* is a mutable, distributed data structure stored entirely inside a git repository
with no external database. Changes are represented as an ordered sequence of *operations*
grouped in *operation packs*. Each operation pack is embedded in a git commit; the commits form a
directed acyclic graph (DAG). Merging two diverged histories produces a merge commit with
an empty operation pack.

The final state of an entity (its *snapshot*) is derived by replaying all operations in a
deterministic order defined by Lamport logical clocks.


## 2. Git references

| Reference pattern                               | Meaning                |
|-------------------------------------------------|------------------------|
| `refs/<namespace>/<entity-id>`                  | Local entity           |
| `refs/remotes/<remote>/<namespace>/<entity-id>` | Remote-tracking entity |

`<namespace>` is a short plural noun chosen by each entity type (e.g. `bugs`).

`<entity-id>` is the 64-character lowercase hex SHA-256 ID of the entity (see [§7](#7-id-derivation)).


## 3. Commit structure

Each commit in the entity's history points to a git tree with the following entries. All
entries whose value is a metadata label (clocks, version) point to the **empty blob**
(`e69de29bb2d1d6434b8b29ae775ad8c2e48c5391`); the information is encoded in the entry
name, not in the blob content. git-bug writes this blob with `git hash-object` on every
commit (equivalent to storing an empty byte sequence); git deduplicates it automatically so
only one copy ever exists in the repository. Because all metadata entries share the same
object, no additional data is transferred when pushing or fetching — any repository that
has ever stored a git-bug entity already has it.

| Tree entry name    | Object type  | Required                              | Meaning                                                                             |
|--------------------|--------------|---------------------------------------|-------------------------------------------------------------------------------------|
| `version-<N>`      | blob (empty) | always                                | Format version; `N` is a decimal integer                                            |
| `ops`              | blob         | always                                | JSON-serialized operation pack (see [§5](#5-operation-pack-blob))                   |
| `edit-clock-<N>`   | blob (empty) | always                                | Lamport edit time; `N` is a decimal integer                                         |
| `create-clock-<N>` | blob (empty) | first commit only                     | Lamport creation time; `N` is a decimal integer                                     |
| `extra/`           | tree         | when operations have file attachments | File blobs referenced by operations in this pack (see [§6.3](#63-file-attachments)) |

The `version-<N>` entry encodes the format version number for this entity type. A reader
**must** check the version before attempting to decode the `ops` blob, and **must** return
a clear error if the version is unrecognized rather than attempting partial decoding.


## 4. DAG topology constraints

A reader **must** reject an entity that violates any of the following:

- The DAG has exactly one root commit (a commit with no parents).
- Non-root commits have at least one parent.
- Merge commits (more than one parent) carry zero operations in their `ops` blob.
- The first (root) commit has a `create-clock-<N>` entry with N > 0.
- Every commit has an `edit-clock-<N>` entry.
- Every commit's edit-clock is strictly greater than every one of its parents' edit-clocks.
- For non-merge commits, the edit-clock **must not** exceed the parent's edit-clock by more
  than 1,000,000 (guard against clock-spam attacks). Merge commits are exempt from this
  limit to allow merging two diverged histories with a large clock gap between them.


## 5. Operation pack blob

The `ops` blob is a JSON object with the following top-level fields:

```json
{
  "author": { "id": "<identity-id>" },
  "ops": [ <operation>, ... ]
}
```

| Field    | Type   | Required | Description                                                        |
|----------|--------|----------|--------------------------------------------------------------------|
| `author` | object | yes      | Identity stub: an object with a single `id` field (64-char hex)    |
| `ops`    | array  | yes      | Ordered array of operation objects; may be empty for merge commits |

The author's full data lives in the identity store (see the identity spec). The `id` here
is a reference into that store.

**All operations in a single pack share the same author.** A reader **must** reject a pack
where any operation's author differs from the pack-level author.

### 5.1 Pack ID

The pack's ID is the SHA-256 of the exact bytes written to the `ops` blob (see [§7](#7-id-derivation)).

### 5.2 Entity ID

The entity's ID is the pack ID of its first (root) commit's operation pack.


## 6. Operations

### 6.1 Common fields (present in every operation)

Every operation is a JSON object with the following fields:

| Field     | JSON key    | Type                   | Required | Description                                           |
|-----------|-------------|------------------------|----------|-------------------------------------------------------|
| Type      | `type`      | integer                | yes      | Operation type identifier; values are entity-specific |
| Timestamp | `timestamp` | integer                | yes      | Unix timestamp (seconds since epoch) for display only |
| Nonce     | `nonce`     | base64 string          | yes      | Random bytes (20–64 bytes) to ensure ID uniqueness    |
| Metadata  | `metadata`  | object (string→string) | no       | Arbitrary key/value pairs (bridge metadata, etc.)     |

`author` and `id` are **not** stored in the operation JSON. The author comes from the
enclosing pack; the ID is derived from the serialized bytes (see [§7](#7-id-derivation)).

A reader **must** treat the `timestamp` value as informational only — it is not verified and
may be inaccurate.

### 6.2 Operation ID

The ID of an operation is the SHA-256 of the exact raw JSON bytes of that operation as they
appear in the `ops` array (see [§7](#7-id-derivation)).

### 6.3 File attachments

Operations that reference binary files (e.g. images in comments) store the git blob hashes
of those files directly in their own JSON fields (e.g. a `files` array). The operation's
content — such as a comment message — may reference a file by its blob hash or by the name
of its entry in the `extra/` subtree (e.g. a filename). All referenced blobs for a given
pack are collected in the `extra/` subtree so that a standard `git fetch` transfers them
automatically alongside the pack. The current implementation uses arbitrary names
(`file0`, `file1`, …) and references files by hash only.

### 6.4 Unknown operation types

A conforming reader **must** accept and preserve operation objects whose `type` value is not
recognized. Unknown operations **must not** affect the snapshot state; they **must** be
retained verbatim if the entity is subsequently written back.

### 6.5 SetMetadataOperation (generic pattern)

`SetMetadataOperation` is a generic operation type defined in the DAG layer and usable by
any entity. Its concrete `type` integer is assigned by each entity (see the entity-specific
spec for the value).

It targets another operation by ID and annotates it with additional key/value metadata.
Metadata values set by this operation are immutable: a key set once cannot be overridden by
a later `SetMetadataOperation`.

| Field               | JSON key       | Type                   | Required |
|---------------------|----------------|------------------------|----------|
| (common fields)     | —              | —                      | yes      |
| Target operation ID | `target`       | 64-char hex string     | yes      |
| Metadata to add     | `new_metadata` | object (string→string) | yes      |

### 6.6 NoOpOperation (generic pattern)

`NoOpOperation` is a generic operation type defined in the DAG layer. It carries no
entity-specific fields beyond the common ones and does not change the snapshot. It is used
by bridges and other tools to store metadata in the history without affecting the entity
state. Its concrete `type` integer is assigned by each entity.


## 7. ID derivation

All IDs in git-bug are 64-character lowercase hex strings derived from SHA-256:

```
id = hex(sha256(data))
```

| What                            | Input data                                                          |
|---------------------------------|---------------------------------------------------------------------|
| Operation ID                    | Raw JSON bytes of the operation object as stored in the `ops` array |
| Pack ID / Entity ID (root pack) | Raw JSON bytes of the full `ops` blob                               |

A reader derives the ID by hashing the bytes exactly as stored in git — no re-encoding is
needed or performed. The reference implementation writes compact JSON (no added whitespace or
pretty print) as a storage convention.


## 8. Lamport clocks

git-bug uses [Lamport logical clocks](https://en.wikipedia.org/wiki/Lamport_timestamps) to
impose a causal order on entities across repositories.

Each entity type maintains two clocks:

| Clock          | Name pattern         | Meaning                                       |
|----------------|----------------------|-----------------------------------------------|
| Creation clock | `<namespace>-create` | Set once, on the entity's first commit        |
| Edit clock     | `<namespace>-edit`   | Incremented on every new commit to the entity |

Clock values are encoded as decimal integers in git tree entry names (e.g.
`create-clock-14`, `edit-clock-137`). The entries point to the empty blob; no data is
stored in blob content.

When reading an entity, a reader **must witness** the clock values it observes. Witnessing
means: if an observed value is greater than the local clock, advance the local clock to
that value. This ensures that new events created after reading are causally ordered after
the read events.


## 9. Operation ordering

The snapshot is built by replaying operations in the following deterministic order:

1. Collect all operation packs across the DAG.
2. Sort packs by `edit-clock` value, **ascending**.
3. For packs with equal edit-clock values (concurrent edits from different machines),
   break ties by **lexicographic order of the pack ID** (SHA-256 hex string, ascending).
   This ordering is arbitrary but stable and hard to manipulate.
4. Within each pack, operations retain their original array order.

The resulting total order uses clock values only — no DAG traversal is required to compute
it. This is safe because clock values are validated against the DAG topology on read
([§4](#4-dag-topology-constraints)): a child commit's clock is always strictly greater than
all its parents' clocks, so the clock ordering is guaranteed to be causally consistent with
the DAG. Concurrent packs (same clock value, different machines) have no causal relationship
by definition, so their relative order is arbitrary but must be stable.


## 10. Merge algorithm

When fetching entity updates from a remote, a client applies one of five scenarios:

| Scenario        | Condition                                                  | Action                                |
|-----------------|------------------------------------------------------------|---------------------------------------|
| 1. New          | Entity does not exist locally                              | Copy the remote reference as-is       |
| 2. Identical    | Local and remote tips are the same commit                  | No action                             |
| 3. Local ahead  | Remote tip is an ancestor of local tip                     | No action                             |
| 4. Fast-forward | Local tip is an ancestor of remote tip                     | Advance local reference to remote tip |
| 5. Diverged     | Neither tip is an ancestor of the other (concurrent edits) | Create a merge commit                 |

**Scenario 5 in detail**: a new commit M is created with two parents (local tip and remote
tip). Its `ops` blob contains an empty operation array. The commit is attributed to the
local user performing the merge and is signed if the user has a signing key. Its
`edit-clock` is the next value from the local edit clock.

**Convergence**: the merge commit has both diverged tips as direct parents, so both become its ancestors.
When any other repo syncs later, the fast-forward check (scenario 4) finds that its tip is
already an ancestor of the merge commit and advances to it without creating another merge commit. In a ring
of repos syncing in a cycle, each round of pulls can only reduce the number of unresolved
divergences; eventually all repos fast-forward to the same tip. The DAG grows monotonically
(git-bug only accepts fast-forward pushes), so a merged state can never be overwritten.

Remote entities fetched via git land at `refs/remotes/<remote>/<namespace>/<id>`. After
merging, the result is stored at `refs/<namespace>/<id>`.


## 11. Signing

If the author of an operation pack has one or more signing keys registered in their
identity at the pack's edit-clock time, the git commit for that pack **must** be signed
with one of those keys using an OpenPGP detached signature.

When reading: if the author has valid keys at the observed edit-clock time, the commit
signature **must** be present and valid. If the author has no keys, unsigned commits are
accepted.

Keys are stored as armored OpenPGP public keys in the identity version that was current at
the time of signing. See the identity spec for key storage details.

**Note**: key management tooling (generating, enrolling, and revoking keys) is under active
development.


## 12. Conformance

A conforming **reader**:

- Validates the git tree structure and rejects entities that violate [§4](#4-dag-topology-constraints).
- Checks the format version before decoding and returns a clear error on mismatch.
- Verifies commit signatures when the author has registered keys ([§11](#11-signing)).
- Accepts and silently skips operation types it does not recognize ([§6.4](#64-unknown-operation-types)).
- Witnesses Lamport clock values ([§8](#8-lamport-clocks)).
- Uses the clock-based operation ordering ([§9](#9-operation-ordering)).
- Ignores entity namespaces it does not implement.

A conforming **writer**:

- Sets a nonce of 20–64 random bytes in every operation.
- Encodes clock values in tree entry names pointing to the empty blob.
- Signs commits when the author has a registered key.
- Does not modify or discard operation packs it does not understand.
- Assigns edit-clock values that are strictly greater than any parent pack's edit-clock,
  and does not jump by more than 1,000,000 from the parent value.

## 13. Test vectors

Test vectors for this layer are in [`testdata/dag-entity.json`](testdata/dag-entity.json).
