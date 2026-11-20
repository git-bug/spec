# Specification: bug entity

This document specifies the operation types for the `bug` entity in git-bug. It assumes
familiarity with the generic DAG entity format described in [dag-entity.md](dag-entity.md).


## 1. Overview

A bug tracks an issue with a title, a body (the opening comment), a status, labels,
assignees, and a thread of comments. It may be part of another bug, its parent, and depend
on other bugs. Its state is derived by replaying an ordered sequence of operations.


## 2. Storage

| Property       | Value                |
|----------------|----------------------|
| Namespace      | `bugs`               |
| Git reference  | `refs/bugs/<bug-id>` |
| Format version | 4                    |

The bug's ID equals the ID of its `CreateOperation`, the first operation of the first pack
(see [ids.md](ids.md#2-derivation)).


## 3. Operation type constants

Operation types are encoded as integers in the `type` field of each operation JSON object.

| Constant             | Integer | Description                                                             |
|----------------------|---------|-------------------------------------------------------------------------|
| `CreateOp`           | 1       | Create the bug (the first operation, and only it)                       |
| `SetTitleOp`         | 2       | Change the bug title                                                    |
| `AddCommentOp`       | 3       | Add a comment                                                           |
| `SetStatusOp`        | 4       | Change the bug status, with the reason when closing                     |
| `LabelChangeOp`      | 5       | Add or remove labels                                                    |
| `EditCommentOp`      | 6       | Edit an existing comment                                                |
| `NoOpOp`             | 7       | No-op (used by bridges to store metadata)                               |
| `SetMetadataOp`      | 8       | Annotate another operation with metadata                                |
| `SetParentOp`        | 9       | Set or clear the parent of the bug                                      |
| `AddDependencyOp`    | 10      | Add a dependency                                                        |
| `RemoveDependencyOp` | 11      | Remove a dependency                                                     |
| `AddAssigneeOp`      | 12      | Add an assignee                                                         |
| `RemoveAssigneeOp`   | 13      | Remove an assignee                                                      |

A reader **must** accept packs containing type values not listed above (see
[dag-entity.md §6.4](dag-entity.md#64-unknown-operation-types)).


## 4. Initial invariant

A bug's first operation **must** be a `CreateOperation` (type 1). A `CreateOperation`
anywhere else is invalid ([dag-entity.md §6](dag-entity.md#6-operations)).


## 5. Operation schemas

All operations include the common fields defined in
[dag-entity.md §6.1](dag-entity.md#61-common-fields-present-in-every-operation).
Only entity-specific fields are listed below. Text, multi-line text and blank text are
defined in [values.md](values.md#5-text), IDs in [ids.md §1](ids.md#1-format), and
git hashes in [dag-entity.md §6.3](dag-entity.md#63-file-attachments). An operation that breaks these rules is
invalid ([dag-entity.md §6](dag-entity.md#6-operations)).

### 5.1 CreateOperation (type 1)

Creates the bug. The ID of this operation becomes the bug's ID.

| Field   | JSON key  | Type                      | Required | Description                                        |
|---------|-----------|---------------------------|----------|----------------------------------------------------|
| Title   | `title`   | string                    | yes      | Bug title; text, not blank                         |
| Message | `message` | string                    | yes      | Opening comment; multi-line text                   |
| Files   | `files`   | array of git hashes       | no       | Blobs referenced in `message`; none if absent      |

**Apply**: initializes all snapshot fields. Sets the bug ID (equal to this operation's ID),
the author, the title, and the first comment. The opening comment is identified by this
operation's ID; its combined ID (see [ids.md](ids.md#5-combined-ids)) is exposed in
APIs and UIs.

Example:
```json
{
  "type": 1,
  "timestamp": 1609459200,
  "nonce": "rv5N8TwqB3sGKhBxVoNFPwAAAAA=",
  "title": "segfault on empty input",
  "message": "Running `./app` with no arguments crashes.",
  "files": []
}
```

### 5.2 SetTitleOperation (type 2)

Changes the bug title.

| Field          | JSON key | Type   | Required | Description                                           |
|----------------|----------|--------|----------|-------------------------------------------------------|
| New title      | `title`  | string | yes      | The new title; text, not blank                        |
| Previous title | `was`    | string | yes      | Title before the change; text                         |

**Apply**: replaces the snapshot title unconditionally. **Last-write-wins**: two concurrent
title changes are resolved by the total operation order
([dag-entity.md §9](dag-entity.md#9-operation-ordering)) — the one that
sorts last wins. The `was` field is informational only and is not validated against the
current snapshot state.

Example:
```json
{
  "type": 2,
  "timestamp": 1609462800,
  "nonce": "9k3mP1QrX8aLuYoW5NcTzAAAAAA=",
  "title": "segfault on empty input in parser",
  "was": "segfault on empty input"
}
```

### 5.3 AddCommentOperation (type 3)

Adds a comment to the bug thread.

| Field   | JSON key  | Type                      | Required | Description                                        |
|---------|-----------|---------------------------|----------|----------------------------------------------------|
| Message | `message` | string                    | yes      | Comment body; multi-line text                      |
| Files   | `files`   | array of git hashes       | no       | Blobs referenced in `message`; none if absent      |

**Apply**: appends a new comment to the snapshot's comment list. The comment is identified
by this operation's ID; its combined ID (see [ids.md](ids.md#5-combined-ids)) is
exposed in APIs and UIs. **Grow-only log semantics**: two concurrent `AddCommentOperation`s
both survive as distinct comments; no comment is ever dropped or overwritten by another.
Their relative order in the list follows the total operation order
([dag-entity.md §9](dag-entity.md#9-operation-ordering)).

Example:
```json
{
  "type": 3,
  "timestamp": 1609466400,
  "nonce": "hWqB2nK7sLmXpTrY4NcVzEAAAAA=",
  "message": "Reproduced on version 1.2.3.",
  "files": []
}
```

### 5.4 SetStatusOperation (type 4)

Changes the bug status. An operation that closes the bug may say why.

| Field        | JSON key       | Type               | Required | Description                                   |
|--------------|----------------|--------------------|----------|-----------------------------------------------|
| Status       | `status`       | integer            | yes      | 1 = open, 2 = closed                          |
| Close reason | `reason`       | integer            | no       | 1 = completed, 2 = not planned, 3 = duplicate |
| Duplicate of | `duplicate_of` | bug ID             | no       | ID of the bug that this one duplicates        |

`reason` goes with `status` 2 only, and `duplicate_of` with `reason` 3 only. A bug may be
closed as a duplicate without `duplicate_of`, when the bug it duplicates is not known. Both
fields are omitted when they are not set.

**Apply**: replaces the snapshot status, close reason and duplicate-of together,
unconditionally: closing without a `reason` leaves the bug with no close reason, and
reopening it clears both. **Last-write-wins**, same as `SetTitleOperation`.

A reader ignores `reason` when `status` is not 2, and reads a `reason` it does not know as
no reason. It ignores `duplicate_of` when `reason` is not 3, or when it is the ID of the bug
itself. `duplicate_of` is a reference to another bug, which may be absent (see
[dag-entity.md §6.8](dag-entity.md#68-references-to-other-entities));
[§8.3](#83-duplicates) defines the original of a duplicate.

A reader that does not know `reason` and `duplicate_of` reads the status alone (see
[dag-entity.md §6.5](dag-entity.md#65-unknown-fields)).

Examples:
```json
{
  "type": 4,
  "timestamp": 1609470000,
  "nonce": "dPx3oR8mN2bQcFyV6LtUwAAAAAA=",
  "status": 2
}
```
```json
{
  "type": 4,
  "timestamp": 1609470000,
  "nonce": "gSw6rY9pN5oMbXzU3VcGaLAAAAA=",
  "status": 2,
  "reason": 3,
  "duplicate_of": "c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2"
}
```

### 5.5 LabelChangeOperation (type 5)

Adds and/or removes labels. A label is text that is not blank, of any length.

| Field          | JSON key  | Type             | Required | Description                      |
|----------------|-----------|------------------|----------|----------------------------------|
| Added labels   | `added`   | array of strings | no       | Labels to add; none if absent    |
| Removed labels | `removed` | array of strings | no       | Labels to remove; none if absent |

At least one of `added` or `removed` must be non-empty.

**Apply**: updates the label set — adds each label in `added` if not already present,
removes each label in `removed` if present (both are idempotent). The set is sorted
lexicographically after each update. **Set semantics with last-write-wins conflict
resolution per label**: if two concurrent operations add and remove the same label, the one
that sorts last in the total operation order determines the final state. A label in `added`
that the repository config doesn't allow at the time of the operation is not added (see
[config.md §9.3](config.md#93-allowed-labels)).

Example:
```json
{
  "type": 5,
  "timestamp": 1609473600,
  "nonce": "eQw4pT9nL3mKbVxR7UcFyZAAAAA=",
  "added": ["bug", "crash"],
  "removed": []
}
```

### 5.6 EditCommentOperation (type 6)

Edits the body of an existing comment (including the opening comment created by
`CreateOperation`).

| Field   | JSON key  | Type                      | Required | Description                                                |
|---------|-----------|---------------------------|----------|------------------------------------------------------------|
| Target  | `target`  | operation ID              | yes      | ID of the operation that originally created the comment    |
| Message | `message` | string                    | yes      | New comment body; multi-line text                          |
| Files   | `files`   | array of git hashes       | no       | Blobs referenced in the new `message`; none if absent      |

**Apply**: locates the comment whose creating operation ID matches `target` and scans the
timeline. If not found, the operation is a silent no-op. If found, the
comment's `message` and `files` in `snapshot.Comments` are replaced, and the new version is
appended to that comment's edit history in the timeline (the full history is preserved, only
the latest version is exposed as the current comment body). **Last-write-wins**: concurrent
edits to the same comment are resolved by total operation order. Any author may edit any
comment (no author restriction is enforced yet).

Example:
```json
{
  "type": 6,
  "timestamp": 1609477200,
  "nonce": "fRv5qX8oM4nLaWyT2UbFzKAAAAA=",
  "target": "a3b4c5d6e7f8a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a1b2c3d4e5f6a7b8",
  "message": "Reproduced on versions 1.2.3 and 1.2.4.",
  "files": []
}
```

### 5.7 NoOpOperation (type 7)

No entity-specific fields beyond the common ones. Does not change the bug state. Used by
bridges to record an event in the history without modifying the snapshot.

See [dag-entity.md §6.7](dag-entity.md#67-noopoperation-generic-pattern).

### 5.8 SetMetadataOperation (type 8)

See [dag-entity.md §6.6](dag-entity.md#66-setmetadataoperation-generic-pattern) for the
full semantics. The `target` ID refers to any operation in the bug's history.

| Field        | JSON key       | Type                   | Required |
|--------------|----------------|------------------------|----------|
| Target       | `target`       | operation ID           | yes      |
| New metadata | `new_metadata` | object (string→string) | no       |

**Apply**: scans all operations in the snapshot for the one whose ID matches `target`, then
merges `new_metadata` into its metadata map. **First-write-wins per key**: if a key is
already present (set by the operation's inline `metadata` or a prior `SetMetadataOperation`
in the total order), the new value is silently discarded. This means the first annotation
that arrives in operation order persists permanently and cannot be overridden.

### 5.9 SetParentOperation (type 9)

Sets or clears the parent of the bug: the bug it is part of.

| Field  | JSON key | Type            | Required | Description                                  |
|--------|----------|-----------------|----------|----------------------------------------------|
| Parent | `parent` | bug ID, or `""` | yes      | ID of the parent bug, or `""` for no parent  |

**Apply**: replaces the snapshot parent unconditionally, so that a bug has at most one
parent; `""` clears it. **Last-write-wins**, same as `SetTitleOperation`. An operation whose `parent` is the
ID of the bug itself does nothing.

The parent is a reference to another bug, which may be absent (see
[dag-entity.md §6.8](dag-entity.md#68-references-to-other-entities)). The parents of several
bugs may form a cycle, which readers break as [§8.1](#81-parent-and-children) defines.

Example:
```json
{
  "type": 9,
  "timestamp": 1609480800,
  "nonce": "hTx7sZ0qO6pNcYaV4WdHbMAAAAA=",
  "parent": "c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2"
}
```

### 5.10 AddDependencyOperation (type 10)

Adds a dependency: a bug that must be finished before this one can proceed.

| Field      | JSON key     | Type               | Required | Description                       |
|------------|--------------|--------------------|----------|-----------------------------------|
| Dependency | `dependency` | bug ID             | yes      | ID of the bug this one depends on |

**Apply**: adds the bug to the snapshot's set of dependencies, if not already present. **Set
semantics with last-write-wins conflict resolution per dependency**, same as the labels of
`LabelChangeOperation`: if two concurrent operations add and remove the same dependency, the
one that sorts last in the total operation order determines the final state. An operation
whose `dependency` is the ID of the bug itself does nothing.

The dependency is a reference to another bug, which may be absent (see
[dag-entity.md §6.8](dag-entity.md#68-references-to-other-entities)).
[§8.2](#82-dependencies) defines what follows from the dependencies of the bugs present.

Example:
```json
{
  "type": 10,
  "timestamp": 1609484400,
  "nonce": "iUy8tA1rP7qOdZbW5XeIcNAAAAA=",
  "dependency": "c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2"
}
```

### 5.11 RemoveDependencyOperation (type 11)

Removes a dependency.

| Field      | JSON key     | Type               | Required | Description                                 |
|------------|--------------|--------------------|----------|---------------------------------------------|
| Dependency | `dependency` | bug ID             | yes      | ID of the bug this one no longer depends on |

**Apply**: removes the bug from the snapshot's set of dependencies, if present. Concurrent
changes resolve as for `AddDependencyOperation`.

### 5.12 AddAssigneeOperation (type 12)

Adds an assignee: an identity in charge of the bug. A bug may have several.

| Field    | JSON key   | Type               | Required | Description        |
|----------|------------|--------------------|----------|--------------------|
| Assignee | `assignee` | identity ID        | yes      | ID of the identity |

**Apply**: adds the identity to the snapshot's set of assignees, if not already present.
**Set semantics with last-write-wins conflict resolution per assignee**, same as for
`AddDependencyOperation`.

The assignee is a reference to an identity, which may be absent (see
[dag-entity.md §6.8](dag-entity.md#68-references-to-other-entities)).

Example:
```json
{
  "type": 12,
  "timestamp": 1609488000,
  "nonce": "jVz9uB2sQ8rPeAcX6YfJdOAAAAA=",
  "assignee": "b29b7b2e0acb0a91dcf3c6c9d8e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7"
}
```

### 5.13 RemoveAssigneeOperation (type 13)

Removes an assignee.

| Field    | JSON key   | Type               | Required | Description        |
|----------|------------|--------------------|----------|--------------------|
| Assignee | `assignee` | identity ID        | yes      | ID of the identity |

**Apply**: removes the identity from the snapshot's set of assignees, if present. Concurrent
changes resolve as for `AddAssigneeOperation`.


## 6. Operation metadata

The `metadata` field present on every operation (see
[dag-entity.md §6.1](dag-entity.md#61-common-fields-present-in-every-operation)) is used
primarily by bridges to record the correspondence between a git-bug operation and its
counterpart on a remote issue tracker.

### 6.1 Well-known keys

| Key                 | Set by           | Applies to        | Description                                                                 |
|---------------------|------------------|-------------------|-----------------------------------------------------------------------------|
| `origin`            | all bridges      | `CreateOperation` | Name of the bridge that imported this bug (e.g. `github`, `gitlab`, `jira`) |
| `github-id`         | GitHub bridge    | any operation     | GitHub node ID of the corresponding issue, comment, or event                |
| `github-url`        | GitHub bridge    | `CreateOperation` | URL of the corresponding GitHub issue                                       |
| `gitlab-id`         | GitLab bridge    | any operation     | GitLab internal ID of the corresponding issue or note                       |
| `gitlab-url`        | GitLab bridge    | `CreateOperation` | URL of the corresponding GitLab issue                                       |
| `gitlab-base-url`   | GitLab bridge    | `CreateOperation` | Base URL of the GitLab instance (for self-hosted)                           |
| `gitlab-project-id` | GitLab bridge    | `CreateOperation` | GitLab project ID                                                           |
| `jira-id`           | Jira bridge      | any operation     | Jira internal ID of the issue or comment                                    |
| `jira-key`          | Jira bridge      | `CreateOperation` | Jira issue key (e.g. `PROJECT-123`)                                         |
| `jira-base-url`     | Jira bridge      | `CreateOperation` | Base URL of the Jira instance                                               |
| `jira-project`      | Jira bridge      | `CreateOperation` | Jira project key                                                            |
| `jira-export-time`  | Jira bridge      | any operation     | Timestamp of the last export of this operation to Jira                      |
| `launchpad-id`      | Launchpad bridge | `CreateOperation` | Launchpad bug ID                                                            |

The `origin` key on the `CreateOperation` is the standard way to identify which bridge
imported a bug. It is used during re-import to match a remote issue back to an existing
local bug.

### 6.2 Inline metadata vs. SetMetadataOperation

Metadata can be attached to an operation in two ways:

- **Inline** (`metadata` field in the operation JSON): set at import time, included in the
  operation's serialized bytes, and therefore part of the operation's ID. This is the
  normal case for bridge import data.
- **Via `SetMetadataOperation`** (type 8): added after the fact, without changing the target
  operation's ID. Used when the metadata is only available after the operation has been
  committed — for example, a remote ID returned by an issue tracker after export.


## 7. Snapshot semantics

The snapshot is built by applying operations in the order defined in
[dag-entity.md §9](dag-entity.md#9-operation-ordering). The resulting state is:


| Property     | Type                                         | Set by                                                               |
|--------------|----------------------------------------------|----------------------------------------------------------------------|
| ID           | 64-char hex                                  | Derived from `CreateOperation` ID                                    |
| Author       | identity                                     | `CreateOperation` author                                             |
| Title        | string                                       | Last `SetTitleOperation`; initially from `CreateOperation`           |
| Status       | open \| closed                               | Last `SetStatusOperation`; initially open                            |
| Close reason | close reason, or none                        | Last `SetStatusOperation`; initially none                            |
| Duplicate of | bug ID, or none                              | Last `SetStatusOperation`; initially none                            |
| Labels       | sorted set of strings                        | Accumulated `LabelChangeOperation`s                                  |
| Assignees    | sorted set of identity IDs                   | Accumulated `AddAssigneeOperation`s, `RemoveAssigneeOperation`s      |
| Parent       | bug ID, or none                              | Last `SetParentOperation`; initially none                            |
| Dependencies | sorted set of bug IDs                        | Accumulated `AddDependencyOperation`s, `RemoveDependencyOperation`s  |
| Comments     | ordered list                                 | `CreateOperation` + `AddCommentOperation`s, with edits applied       |
| Actors       | set of identities                            | All operation authors                                                |
| Participants | set of identities                            | Authors of `CreateOperation` and `AddCommentOperation`s              |


## 8. Relations between bugs

The parent, the dependencies and the duplicate-of of a bug are references to other bugs
(see [dag-entity.md §6.8](dag-entity.md#68-references-to-other-entities)). The snapshot of
a bug holds them as its own operations set them, whether or not the bugs they refer to are
present, and whatever those bugs hold.

| Relation     | Stored on     | Says                                           | Derived from it       |
|--------------|---------------|------------------------------------------------|-----------------------|
| Parent       | The child     | The bug is part of its parent                  | Children              |
| Dependency   | The dependent | The dependency must be finished before the bug | Dependents, readiness |
| Duplicate of | The duplicate | The bug repeats another one                    | Original              |

This section defines what is derived from the snapshots of the bugs present in the
repository, so that every client derives the same. None of it is stored.

### 8.1 Parent and children

A parent gives scope: the bug is one part of the work that its parent describes. It says
nothing about order, which is what dependencies are for ([§8.2](#82-dependencies)).

The **children** of a bug are the bugs present whose parent it is.

As each bug is replayed on its own, concurrent `SetParentOperation`s on different bugs can
make a bug its own ancestor: a cycle that none of them makes alone. A reader **must** break
every cycle the same way:

1. For each bug of the cycle, take the `SetParentOperation` that set its parent: the last
   one in its total operation order.
2. Among these, take the latest: the one whose pack has the highest edit-clock, then the
   highest pack ID.
3. The bug of that operation is read as having no parent, and so as the child of no bug.

A bug has at most one parent, so two cycles never share a bug, and taking one parent away
breaks each. The snapshot of the bug is unchanged: its parent applies again once the cycle
is gone, for example when another bug of the cycle gets a new parent. A client **should**
show that the parent of this bug is set but not applied.

A cycle is only seen, and broken, among the bugs present.

### 8.2 Dependencies

A dependency gives order: it must be finished before the bug can proceed. It says nothing
about scope.

The **dependents** of a bug are the bugs present that have it as a dependency.

A bug is **ready** when it is open and all its dependencies are closed, whatever their
close reason. A dependency that is absent is not known to be closed: a bug that has one is
not ready.

Dependencies may form a cycle, in which no bug is ready as long as all of them are open.
Unlike a cycle of parents, it is kept as it is: a reader **must not** leave a dependency out
to break it. A client **should** report it.

### 8.3 Duplicates

A bug whose close reason is *duplicate*, and that has a duplicate-of, repeats that other
bug, which may itself be the duplicate of a third. The **original** of a duplicate is found
by following the duplicate-of from bug to bug, until a bug that has none, or that is absent:
this bug is the original.

When this walk comes back to a bug it already went through, bugs are duplicates of each
other in a loop. The original of every bug of the loop, and of every bug leading to it, is
then the bug of the loop that was created first: the one with the lowest create-clock (see
[dag-entity.md §8](dag-entity.md#8-lamport-clocks)), then the lowest ID. That bug is read as
an original, not as a duplicate.


## 9. Test vectors

Test vectors for this entity are in [`testdata/bug.json`](testdata/bug.json).
