# Specification: identity format

This document is the formal specification for the identity data format used by git-bug.
Identities represent users who author operations. They are stored independently from
entities and are referenced by ID from operation packs.


## 1. Overview

An identity is a mutable record of a user's profile over time. It is modelled as a linear
sequence of *versions*, each capturing the user's name, contact details, and cryptographic
keys at a point in time. Versions are immutable once written; updating an identity appends
a new version.

The identity format is distinct from the generic DAG entity format: it uses a simple linear
commit chain rather than a DAG, and it does not use operation packs.

**Current format version**: 2


## 2. Git references

| Reference pattern                                | Meaning                  |
|--------------------------------------------------|--------------------------|
| `refs/identities/<identity-id>`                  | Local identity           |
| `refs/remotes/<remote>/identities/<identity-id>` | Remote-tracking identity |

`<identity-id>` is the 64-character lowercase hex SHA-256 ID of the identity (see [§5](#5-id-derivation)).


## 3. Commit and tree structure

Each commit in the identity's linear chain points to a git tree with **exactly one entry**:

| Tree entry name | Object type | Description                                                  |
|-----------------|-------------|--------------------------------------------------------------|
| `version`       | blob        | JSON-serialized identity version (see [§4](#4-version-blob)) |

A reader **must** reject an identity commit whose tree has any entry count other than one,
or whose single entry is not named `version`.

The chain is linear: each commit has at most one parent. Concurrent edits that cannot be
fast-forward merged result in a conflict (see [§6](#6-merge)).


## 4. Version blob

Each `version` blob is a JSON object with the following fields:

| Field          | JSON key     | Type                    | Required | Description                                                                                                 |
|----------------|--------------|-------------------------|----------|-------------------------------------------------------------------------------------------------------------|
| Format version | `version`    | integer                 | yes      | Must equal 2                                                                                                |
| Name           | `name`       | string                  | no       | Display name of the user                                                                                    |
| Email          | `email`      | string                  | no       | Email address (from git config or bridge)                                                                   |
| Login          | `login`      | string                  | no       | Username from a bridge (e.g. GitHub login)                                                                  |
| Avatar URL     | `avatar_url` | string                  | no       | URL to a profile picture                                                                                    |
| Public keys    | `pub_keys`   | array of key objects    | no       | PGP public keys valid from this version onward                                                              |
| Lamport times  | `times`      | object (string→integer) | yes      | Snapshot of all known clock values at the time this version was created (see [§4.1](#41-lamport-times-map)) |
| Unix timestamp | `unix_time`  | integer                 | yes      | Wall-clock time of version creation (seconds since epoch)                                                   |
| Nonce          | `nonce`      | base64 string           | yes      | Random bytes (20–64 bytes) to ensure ID uniqueness of the first version                                     |
| Metadata       | `metadata`   | object (string→string)  | no       | Arbitrary key/value pairs                                                                                   |

A reader **must** reject a version whose `version` field does not equal 2.

### 4.1 Lamport times map

The `times` map records the Lamport clock values of all entity types at the moment the
identity version was created. Keys follow the pattern `<namespace>-create` and
`<namespace>-edit` (e.g. `bugs-create`, `bugs-edit`). This allows identity versions to be
temporally ordered with respect to other entities.

Example — a repository that has the `bugs` entity type, where 14 bugs have been created
and the most recent edit clock is 137:

```json
"times": {
  "bugs-create": 14,
  "bugs-edit": 137
}
```

As new entity types are added to a repository (e.g. `prs`, `boards`), their clocks appear
here too. A reader must tolerate unknown keys in this map.

### 4.2 Key objects and key lifecycle

> [!WARNING]
> Key management is not yet fully operational. The data structures and signature
> verification logic for entity commits are in place, but the tooling to generate, enroll,
> and manage keys is incomplete. In practice, most identities carry no keys and entity
> commits are unsigned. The key format (currently OpenPGP) may also change.

Each entry in `pub_keys` is a JSON string containing an ASCII-armored OpenPGP public key
(PEM block type `PGP PUBLIC KEY BLOCK`). The creation time field in the OpenPGP key
**must** be ignored by readers.

**The `pub_keys` array declares the complete set of valid keys for this version onward.**
It is not a delta from the previous version. To add a key, write a new version whose
`pub_keys` contains all previously valid keys plus the new one. To revoke a key, write a
new version whose `pub_keys` omits it. An empty array means no keys are valid from this
version onward.

This model supports multiple keys simultaneously (e.g. one per device) as well as key
rotation: a user can add a new key before revoking the old one to avoid a gap in signing
capability.

#### Key validity over time

Because operations are authored at specific Lamport clock times, the key set used to verify
a signature must be the one that was valid **when the operation was authored**, not the
current key set. The lookup algorithm is:

1. Walk the identity versions in chronological order.
2. For each version, read its `times[<namespace>-edit]` value.
3. Return the `pub_keys` from the latest version whose clock value ≤ the operation's edit
   clock.

This means a revoked key remains valid for verifying operations that were signed before the
revocation.

#### Identity protection (not yet implemented)

The intended design is that once an identity has at least one key declared, new versions
appended to its chain must be signed by one of the currently valid keys — preventing an
attacker with write access to the repository from silently adding a rogue key or replacing
the key set. This is not implemented: identity version commits are currently unsigned and
no such check is performed on read.

### 4.3 Metadata

The `metadata` field is an arbitrary string key/value store, used primarily by bridges to
record the user's identity on a remote platform. Keys follow the convention
`<bridge-name>-<field>`.

Well-known keys set by the built-in bridges:

| Key               | Set by           | Description                                         |
|-------------------|------------------|-----------------------------------------------------|
| `github-login`    | GitHub bridge    | The user's GitHub username                          |
| `gitlab-login`    | GitLab bridge    | The user's GitLab username                          |
| `jira-login`      | Jira bridge      | The user's Jira username                            |
| `jira-user`       | Jira bridge      | The user's Jira user key (internal Jira identifier) |
| `launchpad-login` | Launchpad bridge | The user's Launchpad username                       |

Third parties adding new bridge-specific keys should prefix them with their bridge name to
avoid collisions.

### 4.4 Example version blobs

Minimal version (no keys, no bridge fields):
```json
{
  "version": 2,
  "times": {"bugs-create": 3, "bugs-edit": 7},
  "unix_time": 1609459200,
  "name": "Alice",
  "email": "alice@example.com",
  "nonce": "rv5N8TwqB3sGKhBxVoNFPw=="
}
```

Version with a public key and a bridge login:
```json
{
  "version": 2,
  "times": {"bugs-create": 5, "bugs-edit": 12},
  "unix_time": 1612137600,
  "name": "Alice",
  "email": "alice@example.com",
  "login": "alice-gh",
  "pub_keys": [
    "-----BEGIN PGP PUBLIC KEY BLOCK-----\n...\n-----END PGP PUBLIC KEY BLOCK-----\n"
  ],
  "nonce": "9k3mP1QrX8aLuYoW5NcTzA=="
}
```

Fields absent from the JSON (`login`, `avatar_url`, `pub_keys`, `metadata`) are omitted
when empty (`omitempty`). A reader must treat missing optional fields as empty/zero values.


## 5. ID derivation

The identity's ID is the SHA-256 of the exact bytes of the **first version's JSON blob**:

```
identity-id = hex(sha256(first_version_blob_bytes))
```

A reader **must** verify that the ID encoded in the git reference matches the ID derived
from the first version blob, and **must** reject the identity if they differ.

See [dag-entity.md §7](dag-entity.md#7-id-derivation) for the general ID derivation rules.


## 6. Merge

Identity merge is **fast-forward only**.

When a remote identity is fetched:

- If the remote tip is an ancestor of the local tip (or they are identical), no action is
  taken.
- If the local tip is an ancestor of the remote tip, the local reference is advanced to the
  remote tip (fast-forward).
- If neither tip is an ancestor of the other (concurrent edits), the merge **fails** with
  an error. The conflict must be resolved manually.

This policy exists because an identity should be controlled by a single user. Concurrent
edits from two independent repositories are unusual, and silent merging could result in
inconsistent key sets.


## 7. Key lookup for signature verification

To verify a commit signature on an entity pack, a reader looks up the author's identity and
finds the keys valid at the pack's edit-clock time:

1. Read the author's identity versions in order.
2. Find the latest version whose `times[<namespace>-edit]` value is ≤ the pack's edit
   clock.
3. The set of `pub_keys` from that version is the valid key set for verification.


## 8. Known limitations

- **Repository-local identities**: identity IDs are derived from content, not from a global
  registry, so the same person may have different identities in different repositories.
  There is no built-in mechanism to assert cross-repository identity equivalence.

- **No concurrent edit support**: the fast-forward-only merge policy means that if the same
  identity is edited from two repositories without synchronizing first, one of the edits
  will be rejected. Users who edit their identity from multiple machines should synchronize
  before editing.

## 9. Test vectors

Test vectors for this layer are in [`testdata/identity.json`](testdata/identity.json).
