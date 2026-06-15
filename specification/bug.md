# Specification: bug entity

This document specifies the operation types for the `bug` entity in git-bug. It assumes
familiarity with the generic DAG entity format described in [dag-entity.md](dag-entity.md).


## 1. Overview

A bug tracks an issue with a title, a body (the opening comment), a status, labels, and a
thread of comments. Its state is derived by replaying an ordered sequence of operations.


## 2. Storage

| Property       | Value                |
|----------------|----------------------|
| Namespace      | `bugs`               |
| Git reference  | `refs/bugs/<bug-id>` |
| Format version | 4                    |

The bug's ID equals the ID of its first operation pack, which in turn equals the ID of its
`CreateOperation` (the only operation in the first pack).


## 3. Operation type constants

Operation types are encoded as integers in the `type` field of each operation JSON object.

| Constant        | Integer | Description                                                             |
|-----------------|---------|-------------------------------------------------------------------------|
| `CreateOp`      | 1       | Create the bug (must be the first and only operation in the first pack) |
| `SetTitleOp`    | 2       | Change the bug title                                                    |
| `AddCommentOp`  | 3       | Add a comment                                                           |
| `SetStatusOp`   | 4       | Change the bug status                                                   |
| `LabelChangeOp` | 5       | Add or remove labels                                                    |
| `EditCommentOp` | 6       | Edit an existing comment                                                |
| `NoOpOp`        | 7       | No-op (used by bridges to store metadata)                               |
| `SetMetadataOp` | 8       | Annotate another operation with metadata                                |

A reader **must** accept packs containing type values not listed above (see
[dag-entity.md §6.4](dag-entity.md#64-unknown-operation-types)).


## 4. Initial invariant

A valid bug's first operation **must** be a `CreateOperation` (type 1). No other
`CreateOperation` may appear in subsequent operations.


## 5. Operation schemas

All operations include the common fields defined in
[dag-entity.md §6.1](dag-entity.md#61-common-fields-present-in-every-operation).
Only entity-specific fields are listed below.

### 5.1 CreateOperation (type 1)

Creates the bug. The ID of this operation becomes the bug's ID.

| Field   | JSON key  | Type                      | Required | Description                                        |
|---------|-----------|---------------------------|----------|----------------------------------------------------|
| Title   | `title`   | string                    | yes      | Single-line bug title; must not be empty           |
| Message | `message` | string                    | yes      | Opening comment body (may be empty)                |
| Files   | `files`   | array of git hash strings | yes      | Blobs referenced in `message`; empty array if none |

**Apply**: initializes all snapshot fields. Sets the bug ID (equal to this operation's ID),
the author, the title, and the first comment. The opening comment is identified by this
operation's ID; its combined ID (see [README.md](README.md#combined-ids)) is exposed in
APIs and UIs. A second `CreateOperation` encountered during replay is silently ignored.

Example:
```json
{
  "type": 1,
  "timestamp": 1609459200,
  "nonce": "rv5N8TwqB3sGKhBxVoNFPw==",
  "title": "segfault on empty input",
  "message": "Running `./app` with no arguments crashes.",
  "files": []
}
```

### 5.2 SetTitleOperation (type 2)

Changes the bug title.

| Field          | JSON key | Type   | Required | Description                                           |
|----------------|----------|--------|----------|-------------------------------------------------------|
| New title      | `title`  | string | yes      | The new title                                         |
| Previous title | `was`    | string | yes      | The title before this change (for display in history) |

**Apply**: replaces the snapshot title unconditionally. **Last-write-wins**: two concurrent
title changes are resolved by the total operation order (§9 of the DAG spec) — the one that
sorts last wins. The `was` field is informational only and is not validated against the
current snapshot state.

Example:
```json
{
  "type": 2,
  "timestamp": 1609462800,
  "nonce": "9k3mP1QrX8aLuYoW5NcTzA==",
  "title": "segfault on empty input in parser",
  "was": "segfault on empty input"
}
```

### 5.3 AddCommentOperation (type 3)

Adds a comment to the bug thread.

| Field   | JSON key  | Type                      | Required | Description                                        |
|---------|-----------|---------------------------|----------|----------------------------------------------------|
| Message | `message` | string                    | yes      | Comment body (may be empty)                        |
| Files   | `files`   | array of git hash strings | yes      | Blobs referenced in `message`; empty array if none |

**Apply**: appends a new comment to the snapshot's comment list. The comment is identified
by this operation's ID; its combined ID (see [README.md](README.md#combined-ids)) is
exposed in APIs and UIs. **Grow-only log semantics**: two concurrent `AddCommentOperation`s
both survive as distinct comments; no comment is ever dropped or overwritten by another.
Their relative order in the list follows the total operation order (§9 of the DAG spec).

Example:
```json
{
  "type": 3,
  "timestamp": 1609466400,
  "nonce": "hWqB2nK7sLmXpTrY4NcVzE==",
  "message": "Reproduced on version 1.2.3.",
  "files": []
}
```

### 5.4 SetStatusOperation (type 4)

Changes the bug status.

| Field  | JSON key | Type    | Required | Description          |
|--------|----------|---------|----------|----------------------|
| Status | `status` | integer | yes      | 1 = open, 2 = closed |

**Apply**: replaces the snapshot status unconditionally. **Last-write-wins**, same as
`SetTitleOperation`.

Example:
```json
{
  "type": 4,
  "timestamp": 1609470000,
  "nonce": "dPx3oR8mN2bQcFyV6LtUwA==",
  "status": 2
}
```

### 5.5 LabelChangeOperation (type 5)

Adds and/or removes labels. Labels are arbitrary non-empty strings.

| Field          | JSON key  | Type             | Required | Description                    |
|----------------|-----------|------------------|----------|--------------------------------|
| Added labels   | `added`   | array of strings | yes      | Labels to add; may be empty    |
| Removed labels | `removed` | array of strings | yes      | Labels to remove; may be empty |

At least one of `added` or `removed` must be non-empty.

**Apply**: updates the label set — adds each label in `added` if not already present,
removes each label in `removed` if present (both are idempotent). The set is sorted
lexicographically after each update. **Set semantics with last-write-wins conflict
resolution per label**: if two concurrent operations add and remove the same label, the one
that sorts last in the total operation order determines the final state.

Example:
```json
{
  "type": 5,
  "timestamp": 1609473600,
  "nonce": "eQw4pT9nL3mKbVxR7UcFyZ==",
  "added": ["bug", "crash"],
  "removed": []
}
```

### 5.6 EditCommentOperation (type 6)

Edits the body of an existing comment (including the opening comment created by
`CreateOperation`).

| Field   | JSON key  | Type                      | Required | Description                                                |
|---------|-----------|---------------------------|----------|------------------------------------------------------------|
| Target  | `target`  | 64-char hex string        | yes      | ID of the operation that originally created the comment    |
| Message | `message` | string                    | yes      | New comment body                                           |
| Files   | `files`   | array of git hash strings | yes      | Blobs referenced in the new `message`; empty array if none |

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
  "nonce": "fRv5qX8oM4nLaWyT2UbFzK==",
  "target": "a3b4c5d6e7f8a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a1b2c3d4e5f6a7b8",
  "message": "Reproduced on versions 1.2.3 and 1.2.4.",
  "files": []
}
```

### 5.7 NoOpOperation (type 7)

No entity-specific fields beyond the common ones. Does not change the bug state. Used by
bridges to record an event in the history without modifying the snapshot.

See [dag-entity.md §6.6](dag-entity.md#66-noopoperation-generic-pattern).

### 5.8 SetMetadataOperation (type 8)

See [dag-entity.md §6.5](dag-entity.md#65-setmetadataoperation-generic-pattern) for the
full semantics. The `target` ID refers to any operation in the bug's history.

**Apply**: scans all operations in the snapshot for the one whose ID matches `target`, then
merges `new_metadata` into its metadata map. **First-write-wins per key**: if a key is
already present (set by the operation's inline `metadata` or a prior `SetMetadataOperation`
in the total order), the new value is silently discarded. This means the first annotation
that arrives in operation order persists permanently and cannot be overridden.

| Field        | JSON key       | Type                   | Required |
|--------------|----------------|------------------------|----------|
| Target       | `target`       | 64-char hex string     | yes      |
| New metadata | `new_metadata` | object (string→string) | yes      |


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


| Property     | Type                  | Set by                                                         |
|--------------|-----------------------|----------------------------------------------------------------|
| ID           | 64-char hex           | Derived from `CreateOperation` ID                              |
| Author       | identity              | `CreateOperation` author                                       |
| Title        | string                | Last `SetTitleOperation`; initially from `CreateOperation`     |
| Status       | open \| closed        | Last `SetStatusOperation`; initially open                      |
| Labels       | sorted set of strings | Accumulated `LabelChangeOperation`s                            |
| Comments     | ordered list          | `CreateOperation` + `AddCommentOperation`s, with edits applied |
| Actors       | set of identities     | All operation authors                                          |
| Participants | set of identities     | Authors of `CreateOperation` and `AddCommentOperation`s        |

## 8. Test vectors

Test vectors for this entity are in [`testdata/bug.json`](testdata/bug.json).
