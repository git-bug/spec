# Specification: identifiers

This document specifies the identifiers (IDs) used by both the
[DAG entity format](dag-entity.md) and the [identity format](identity.md).

Every ID is the SHA-256 of bytes stored in git. The random `nonce` present in every
operation and identity version makes otherwise identical content hash to different IDs.


## 1. Format

An ID is a **64-character lowercase hex string**:

```
id = lowercase_hex(sha256(data))
```

A reader **must** reject an ID whose length is not 64. A 40-character ID comes from an older
SHA-1 based format; a reader **should** point to
[git-bug-migration](https://github.com/git-bug/git-bug-migration).


## 2. Derivation

An ID is derived from the **exact bytes stored in git**. A reader **must not** decode and
re-encode before hashing. A writer that needs an ID before writing **must** hash the same
bytes it will write.

| ID                  | Derived from                                                         |
|---------------------|----------------------------------------------------------------------|
| Operation ID        | Raw bytes of the operation object, as they appear in the `ops` array |
| Operation pack ID   | Raw bytes of the full `ops` blob                                     |
| **Entity ID**       | ID of the **first operation** of the root commit's pack              |
| **Identity ID**     | Raw bytes of the **first** `version` blob of the identity            |

An operation ID covers all its fields, including inline `metadata`.


## 3. Where IDs appear

| Location                     | ID kind      | Reader check                                                           |
|------------------------------|--------------|------------------------------------------------------------------------|
| `refs/<namespace>/<id>`      | Entity ID    | **Must** match the first operation's ID, else reject                   |
| `refs/identities/<id>`       | Identity ID  | **Must** match the first version's hash, else reject                   |
| `author.id` in an `ops` blob | Identity ID  | **Must** resolve to a known identity                                   |
| `target` of an operation     | Operation ID | Defined by the entity spec                                             |
| Operation ordering tie-break | Pack ID      | Lexicographic ([dag-entity.md §9](dag-entity.md#9-operation-ordering)) |

Operation IDs **must** be unique within an entity; a reader **must** reject an entity with
duplicates.


## 4. Human-readable prefixes

Clients display IDs as **7-character prefixes** and accept prefixes as input. A prefix
resolves if it matches exactly one known ID; a prefix matching several IDs is an error.
Stored data always uses full IDs.


## 5. Combined IDs

Some elements within an entity, such as a comment within a bug, need to be addressed
directly by users and APIs. Such an element is identified by the ID of the operation that
created it, but an operation ID alone does not tell which entity holds it: finding the
element would require loading every entity and searching their operations.

A **combined ID** solves this by carrying both in a single string: it interleaves a
*primary* ID (the entity) and a *secondary* ID (the element). A client resolving it first
narrows the search to entities matching the primary part, then looks for the element only
within those. The interleaving is arranged so that even a short prefix, as typed by a user,
carries enough of both parts to be resolved.

Combined IDs are derived at read time and used by APIs and UIs.

### 5.1 Construction

Characters are taken in order from each source ID following this 64-character pattern
(P = primary, S = secondary):

```
PSPSPSPPPSPPPPSPPPPSPPPPSPPPPSPPPPSPPPPSPPPPSPPPPSPPPPSPPPPSPPPP
```

That is, position `i` (zero-based) comes from the secondary when `i ∈ {1, 3, 5, 9}` or
`i ≥ 10 and i mod 5 = 4`, and from the primary otherwise. The result holds the first 50
characters of the primary and the first 14 of the secondary.

The pattern is front-heavy on the secondary so that short prefixes carry part of both IDs
(see [§6](#6-combined-id-prefix-collisions)).

### 5.2 Resolution

1. Split the prefix into primary and secondary prefixes using the same position rule.
2. Find the entities whose ID starts with the primary prefix.
3. Among their elements, find those whose combined ID starts with the full prefix.
4. Succeed on exactly one match; otherwise report ambiguity or absence.

### 5.3 Usage

| Entity | Element                                         | Primary | Secondary                       |
|--------|-------------------------------------------------|---------|---------------------------------|
| `bug`  | Comment (including the opening one)             | Bug ID  | ID of the creating operation    |
| `bug`  | Timeline item (title, status, label changes)    | Bug ID  | ID of the producing operation   |


## 6. Combined ID prefix collisions

For each prefix length, the table below gives the number of characters taken from each ID,
and the number of entities (resp. elements within one entity) at which two of them have a
~50% chance of sharing that part of the prefix (birthday bound, ≈ 1.18 · √(16^chars)):

| Prefix length | Primary chars | Secondary chars | Entities for 50% primary collision | Elements per entity for 50% secondary collision |
|---------------|---------------|-----------------|------------------------------------|-------------------------------------------------|
| 5             | 3             | 2               | 75                                 | 19                                              |
| 6             | 3             | 3               | 75                                 | 75                                              |
| 7             | 4             | 3               | 300                                | 75                                              |
| 8             | 5             | 3               | 1,200                              | 75                                              |
| 9             | 6             | 3               | 4,800                              | 75                                              |
| 10            | 6             | 4               | 4,800                              | 300                                             |
| 11            | 7             | 4               | 19,000                             | 300                                             |
| 12            | 8             | 4               | 77,000                             | 300                                             |
| 13            | 9             | 4               | 310,000                            | 300                                             |
| 14            | 10            | 4               | 1.2 × 10^6                         | 300                                             |
| 15            | 10            | 5               | 1.2 × 10^6                         | 1,200                                           |
| 16            | 11            | 5               | 4.9 × 10^6                         | 1,200                                           |
| 24            | 18            | 6               | 8.1 × 10^10                        | 4,800                                           |
| 32            | 24            | 8               | 3.3 × 10^14                        | 77,000                                          |
| 64 (full)     | 50            | 14              | 1.5 × 10^30                        | 3.2 × 10^8                                      |

A prefix is ambiguous only when another element matches it entirely. For a prefix of length
`n` holding `s` secondary characters, in a repository of `E` entities with `M` elements
each, the probability that the prefix of a given element is ambiguous is approximately:

```
P ≈ (M − 1) / 16^s  +  (E − 1) · M / 16^n
```

The first term counts elements of the same entity, which share the primary part and collide
on the secondary part. The second counts elements of other entities, which must collide on
the whole prefix. The first term dominates, so resolving power mostly depends on the number
of secondary characters and of elements per entity.

| Prefix length | E = 100, M = 10 | E = 1,000, M = 20 | E = 10,000, M = 50 |
|---------------|-----------------|-------------------|--------------------|
| 5             | 3.5%            | 8.9%              | 49%                |
| 6             | 0.23%           | 0.58%             | 4.1%               |
| 7             | 0.22%           | 0.47%             | 1.4%               |
| 8             | 0.22%           | 0.46%             | 1.2%               |
| 9             | 0.22%           | 0.46%             | 1.2%               |
| 10            | 0.014%          | 0.029%            | 0.075%             |
| 11            | 0.014%          | 0.029%            | 0.075%             |
| 12            | 0.014%          | 0.029%            | 0.075%             |
| 13            | 0.014%          | 0.029%            | 0.075%             |
| 14            | 0.014%          | 0.029%            | 0.075%             |
| 15            | 0.00086%        | 0.0018%           | 0.0047%            |
| 16            | 0.00086%        | 0.0018%           | 0.0047%            |
| 24            | 5.4 × 10^-5%    | 1.1 × 10^-4%      | 2.9 × 10^-4%       |
| 32            | 2.1 × 10^-7%    | 4.4 × 10^-7%      | 1.1 × 10^-6%       |
| 64 (full)     | 1.2 × 10^-14%   | 2.6 × 10^-14%     | 6.8 × 10^-14%      |

An ambiguous prefix is resolved by a longer one.


## 7. Test vectors

See `id_derivation` in [`testdata/dag-entity.json`](testdata/dag-entity.json),
[`testdata/identity.json`](testdata/identity.json) and [`testdata/bug.json`](testdata/bug.json).
