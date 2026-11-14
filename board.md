# Specification: board entity

This document specifies the `board` entity in git-bug: its columns, its items, and the
operation types that arrange them. It assumes familiarity with the generic DAG entity format
described in [dag-entity.md](dag-entity.md).


## 1. Overview

A board arranges bugs and drafts into columns, to plan work or to follow a process. Its
state is derived by replaying an ordered sequence of operations.

What a board says about a bug is stored on the board: the column the bug is in, and its
place among the other items. Several boards may hold the same bug and arrange it
differently, and putting a bug on a board does not change the bug.


## 2. Storage

| Property       | Value                          |
|----------------|--------------------------------|
| Namespace      | `boards`                       |
| Git reference  | `refs/boards/<board-id>`       |
| Format version | 1                              |
| Clocks         | `boards-create`, `boards-edit` |

The board's ID equals the ID of its `CreateOperation`, the first operation of the first pack
(see [ids.md](ids.md#2-derivation)).


## 3. Model

### 3.1 Columns

A column is a named place of the board, in which items sit. It belongs to its board: the
columns of two boards are unrelated, even under the same name.

A column is identified by the ID of the `AddColumnOperation` that added it. Items refer to
their column by this ID, so that renaming a column changes no item. A column has:

| Property         | Description                                                             |
|------------------|-------------------------------------------------------------------------|
| Name             | Text, not blank. Two columns may have the same name.                    |
| Position         | Orders the columns (see [positions.md](positions.md))                   |
| Priority-ordered | Whether the order of its items is their priority                        |

The columns are listed by position, and columns with the same position by ID.

In a **priority-ordered** column, the order of the items is their priority, the first item
being the one to take first: typically a backlog. In the other columns the order says
nothing, typically for the work under way or done, and a client may list their items
differently, for example the finished work by closing time.

### 3.2 Items

An item is one entry of the board. It is identified by the ID of the operation that added
it, and has:

| Property | Description                                                |
|----------|------------------------------------------------------------|
| Content  | A draft, or a reference to an entity                       |
| Column   | The ID of its column, or none                              |
| Position | Its place in the order of the board ([§3.3](#33-order))    |

A **draft** is a title and nothing more: a note that is not a bug, or not yet. A draft can
be converted into a bug ([§6.12](#612-convertdraftoperation-type-12)); the item then keeps
its ID, its column and its position.

A reference to an entity is made of its namespace and its ID, such as `bugs` and the ID of
a bug. It is a soft reference (see
[dag-entity.md §6.8](dag-entity.md#68-references-to-other-entities)): the item stays on the
board whether or not the entity is present in the repository, and a reader that does not
implement the namespace keeps the item as well. A board holds a given entity at most once:
no two of its items refer to the same entity ID, whatever their namespace.

Operations designate the item they act on by an ID, which is either:

- the ID of the item; or
- for an item that refers to an entity, the ID of that entity.

A writer **must** designate an item that refers to an entity by the ID of the entity. Two
people who add the same bug at the same time then name the same item in what they do next,
whichever of their two operations added it
([§6.8](#68-additementityoperation-type-8)). The ID of the item designates it as well, so
that an operation made on a draft still applies when the draft was converted in the
meantime.

An item without a column is on the board but not placed in it. Clients typically list these
items in a column of their own, before the others: the inbox of the board.

### 3.3 Order

A board has a single order of its items, whatever their column: every item has one
position, and the items are listed by position (see [positions.md](positions.md)), then by
ID for those with the same position. The items of a column are listed in that same order.

A writer places an item between two items of a column by giving it a position between
theirs. The two are neighbours in that column only, and items of other columns may lie
between them in the order of the board: any position between the two will do. As positions
do not depend on the columns, the order of the board still holds when its items are shown
otherwise than by column.

Moving an item sets its position, and its column when the column changes
([§6.9](#69-moveitemoperation-type-9)); no other item is changed. A writer **must** choose
positions as [positions.md §4](positions.md#4-choosing-a-position) requires. Several people
work on a board at the same time: a writer **should** pick positions at random between the
neighbours, and give items that it adds or moves together positions close to each other, as
described there.

### 3.4 Missing targets

A column or an item is *present* from the operation that adds it to the one that removes
it. Any operation may be applied after a concurrent one that removed what it targets. Such
an operation is valid, and applying it never fails:

| When                                                                                | Then                                        |
|-------------------------------------------------------------------------------------|---------------------------------------------|
| An operation designates no item that is present                                     | It does nothing                             |
| `EditDraftOperation` or `ConvertDraftOperation` targets an item that is not a draft | It does nothing                             |
| `EditColumnOperation` or `RemoveColumnOperation` targets a column that is not present | It does nothing                           |
| An item is added or moved to a column that is not present                           | It is added or moved, with no column        |
| A column is removed                                                                 | Its items stay on the board, with no column |
| An entity is added, or a draft converted into an entity, that the board holds       | It does nothing                             |

A removed column or item does not come back. An entity whose item was removed can be added
again, as a new item, which the ID of the entity then designates.


## 4. Operation type constants

Operation types are encoded as integers in the `type` field of each operation JSON object.

| Constant           | Integer | Description                                                           |
|--------------------|---------|-----------------------------------------------------------------------|
| `CreateOp`         | 1       | Create the board (the first operation, and only it)                   |
| `SetTitleOp`       | 2       | Change the board title                                                |
| `SetDescriptionOp` | 3       | Change the board description                                          |
| `AddColumnOp`      | 4       | Add a column                                                          |
| `EditColumnOp`     | 5       | Rename a column, move it, or change whether it is priority-ordered    |
| `RemoveColumnOp`   | 6       | Remove a column                                                       |
| `AddItemDraftOp`   | 7       | Add a draft                                                           |
| `AddItemEntityOp`  | 8       | Add an entity, such as a bug                                          |
| `MoveItemOp`       | 9       | Move an item: change its column, its place in the order, or both      |
| `RemoveItemOp`     | 10      | Remove an item                                                        |
| `EditDraftOp`      | 11      | Change the title of a draft                                           |
| `ConvertDraftOp`   | 12      | Turn a draft into a reference to an entity                            |
| `SetMetadataOp`    | 13      | Annotate another operation with metadata                              |

A reader **must** accept packs containing type values not listed above (see
[dag-entity.md §6.4](dag-entity.md#64-unknown-operation-types)).


## 5. Creation

A board's first operation **must** be a `CreateOperation` (type 1). A `CreateOperation`
anywhere else is invalid ([dag-entity.md §6](dag-entity.md#6-operations)).

Other operations may follow it in the first pack: a board is typically created together
with its columns, each added by an `AddColumnOperation`. A board without any column is
valid, and all its items are then without a column.


## 6. Operation schemas

All operations include the common fields defined in
[dag-entity.md §6.1](dag-entity.md#61-common-fields-present-in-every-operation).
Only entity-specific fields are listed below. Text and blank text are defined in
[values.md](values.md#5-text), and IDs in [ids.md §1](ids.md#1-format).

An operation is invalid ([dag-entity.md §6](dag-entity.md#6-operations)) when, for example,
it has no required field, a value of the wrong type, a blank title or name, or an invalid
position ([positions.md §2](positions.md#2-format)) or ID. An operation whose target is not
present is not invalid ([§3.4](#34-missing-targets)).

### 6.1 CreateOperation (type 1)

Creates the board. The ID of this operation becomes the board's ID.

| Field       | JSON key      | Type   | Required | Description                                |
|-------------|---------------|--------|----------|--------------------------------------------|
| Title       | `title`       | string | yes      | Board title; text, not blank               |
| Description | `description` | string | yes      | Text, may be blank                         |

**Apply**: initializes all snapshot fields. Sets the board ID (equal to this operation's
ID), the title and the description. The board has no column and no item.

Example:
```json
{
  "type": 1,
  "timestamp": 1609459200,
  "nonce": "kWa0vC3tR9sQfBdY7ZgKePAAAAA=",
  "title": "Roadmap",
  "description": "What we work on, and in what order"
}
```

### 6.2 SetTitleOperation (type 2)

Changes the board title.

| Field          | JSON key | Type   | Required | Description                                           |
|----------------|----------|--------|----------|-------------------------------------------------------|
| New title      | `title`  | string | yes      | The new title; text, not blank                        |
| Previous title | `was`    | string | yes      | The title before this change, for display; text       |

**Apply**: replaces the snapshot title unconditionally. **Last-write-wins**: two concurrent
title changes are resolved by the total operation order
([dag-entity.md §9](dag-entity.md#9-operation-ordering)) — the one that sorts last wins. The
`was` field is informational only and is not validated against the current snapshot state.

Example:
```json
{
  "type": 2,
  "timestamp": 1609462800,
  "nonce": "lXb1wD4uS0tRgCeZ8AhLfQAAAAA=",
  "title": "Roadmap 2021",
  "was": "Roadmap"
}
```

### 6.3 SetDescriptionOperation (type 3)

Changes the board description.

| Field                | JSON key      | Type   | Required | Description                                                 |
|----------------------|---------------|--------|----------|-------------------------------------------------------------|
| New description      | `description` | string | yes      | The new description; text, may be blank                     |
| Previous description | `was`         | string | yes      | The description before this change, for display; text       |

**Apply**: replaces the snapshot description unconditionally. **Last-write-wins**, same as
`SetTitleOperation`.

### 6.4 AddColumnOperation (type 4)

Adds a column.

| Field            | JSON key           | Type     | Required | Description                                             |
|------------------|--------------------|----------|----------|---------------------------------------------------------|
| Name             | `name`             | string   | yes      | Text, not blank                                         |
| Position         | `position`         | position | yes      | Place of the column among the others                    |
| Priority-ordered | `priority_ordered` | boolean  | yes      | Whether the column is priority-ordered                  |

**Apply**: adds a column to the snapshot, identified by this operation's ID. Two concurrent
`AddColumnOperation`s both add their column, even under the same name.

Example:
```json
{
  "type": 4,
  "timestamp": 1609459200,
  "nonce": "nZd3yF6wU2vTiEgB0CjNhSAAAAA=",
  "name": "Backlog",
  "position": "a0",
  "priority_ordered": true
}
```

### 6.5 EditColumnOperation (type 5)

Changes a column: its name, its position, whether it is priority-ordered, or several of
these.

| Field            | JSON key           | Type               | Required | Description      |
|------------------|--------------------|--------------------|----------|------------------|
| Column           | `column`           | column ID          | yes      | ID of the column |
| Name             | `name`             | string             | no       | The new name     |
| Position         | `position`         | position           | no       | The new position |
| Priority-ordered | `priority_ordered` | boolean            | no       | The new value    |

At least one of `name`, `position` and `priority_ordered` must be present. A field that is
absent keeps its value.

**Apply**: if the column is not present, does nothing. Otherwise replaces each property
whose field is present. **Last-write-wins for each property**: two concurrent operations
that rename and move the same column both take effect, and of two that rename it, the one
that sorts last in the total operation order wins.

Example:
```json
{
  "type": 5,
  "timestamp": 1609466400,
  "nonce": "oAe4zG7xV3wUjFhC1DkOiTAAAAA=",
  "column": "d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5",
  "name": "Doing"
}
```

### 6.6 RemoveColumnOperation (type 6)

Removes a column.

| Field  | JSON key | Type               | Required | Description      |
|--------|----------|--------------------|----------|------------------|
| Column | `column` | column ID          | yes      | ID of the column |

**Apply**: if the column is not present, does nothing. Otherwise removes the column from
the snapshot. Its items stay on the board with their position, and have no column. The
removal wins over any concurrent operation on the column, whichever sorts last: the column
is removed, and an item moved to it has no column.

### 6.7 AddItemDraftOperation (type 7)

Adds a draft to the board.

| Field    | JSON key   | Type               | Required | Description                                  |
|----------|------------|--------------------|----------|----------------------------------------------|
| Title    | `title`    | string             | yes      | Draft title; text, not blank                 |
| Column   | `column`   | column ID, or `""` | yes      | ID of the column, or `""` for no column      |
| Position | `position` | position           | yes      | Place of the item in the order of the board  |

**Apply**: adds an item to the snapshot, identified by this operation's ID, holding a draft
with this title. If `column` is `""`, or not present on the board, the item has no column.

Example:
```json
{
  "type": 7,
  "timestamp": 1609462800,
  "nonce": "qCg6bI9zX5yWlHjE3FmQkVAAAAA=",
  "title": "Write the release notes",
  "column": "d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5",
  "position": "a1"
}
```

### 6.8 AddItemEntityOperation (type 8)

Adds an entity to the board, such as a bug.

| Field     | JSON key    | Type               | Required | Description                                 |
|-----------|-------------|--------------------|----------|---------------------------------------------|
| Namespace | `namespace` | string             | yes      | Namespace, e.g. `bugs`; text, not blank     |
| Entity    | `entity`    | entity ID          | yes      | ID of the entity                            |
| Column    | `column`    | column ID, or `""` | yes      | ID of the column, or `""` for no column     |
| Position  | `position`  | position           | yes      | Place of the item in the order of the board |

**Apply**: if an item present on the board already refers to an entity with this ID, does
nothing: the board holds the entity once. Otherwise adds an item to the snapshot,
identified by this operation's ID, referring to the entity. If `column` is `""`, or not
present on the board, the item has no column.

Of two concurrent operations adding the same entity, the one that sorts first in the total
operation order adds the item, at the place it gives. What either author does to the entity
next applies to that item, as operations designate it by the ID of the entity
([§3.2](#32-items)).

Example:
```json
{
  "type": 8,
  "timestamp": 1609462800,
  "nonce": "rDh7cJ0aY6zXmIkF4GnRlWAAAAA=",
  "namespace": "bugs",
  "entity": "a3b4c5d6e7f8a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a1b2c3d4e5f6a7b8",
  "column": "d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5",
  "position": "a0"
}
```

### 6.9 MoveItemOperation (type 9)

Moves an item: to another column, to another place in the order of the board, or both.

| Field    | JSON key   | Type                          | Required | Description                                 |
|----------|------------|-------------------------------|----------|---------------------------------------------|
| Item     | `item`     | item or entity ID             | yes      | ID designating the item ([§3.2](#32-items)) |
| Column   | `column`   | column ID, or `""`            | no       | The new column, see below                   |
| Position | `position` | position                      | yes      | Place of the item in the order of the board |

`column` has three forms:

- absent: the item keeps its column;
- the empty string `""`: the item has no column any more;
- an ID: the item goes to that column.

**Apply**: if the item is not present, does nothing. Otherwise replaces its position, and
its column unless `column` is absent. If `column` is the ID of a column that is not present,
the item has no column. **Last-write-wins, for the position and for the column**: of two
concurrent moves of the same item, the one that sorts last in the total operation order
gives the position, and the column comes from the last one that sets a column. Moving an
item within its column therefore does not undo a concurrent move to another column.

Moves of different items never conflict. If they give the same position to two items, these
are listed by ID ([§3.3](#33-order)).

Examples, the first moving an item within its column, the second to another column:
```json
{
  "type": 9,
  "timestamp": 1609466400,
  "nonce": "sEi8dK1bZ7aYnJlG5HoSmXAAAAA=",
  "item": "e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6",
  "position": "a0V"
}
```
```json
{
  "type": 9,
  "timestamp": 1609470000,
  "nonce": "tFj9eL2cA8bZoKmH6IpTnYAAAAA=",
  "item": "e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6",
  "column": "d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5",
  "position": "a2"
}
```

### 6.10 RemoveItemOperation (type 10)

Removes an item from the board. The entity it refers to, if any, is not changed.

| Field | JSON key | Type               | Required | Description                                 |
|-------|----------|--------------------|----------|---------------------------------------------|
| Item  | `item`   | item or entity ID  | yes      | ID designating the item ([§3.2](#32-items)) |

**Apply**: if the item is not present, does nothing. Otherwise removes the item from the
snapshot. The removal wins over any concurrent operation on the item, whichever sorts last.

### 6.11 EditDraftOperation (type 11)

Changes the title of a draft.

| Field | JSON key | Type               | Required | Description                                    |
|-------|----------|--------------------|----------|------------------------------------------------|
| Item  | `item`   | item ID            | yes      | ID of the item                                 |
| Title | `title`  | string             | yes      | The new title; text, not blank                 |

**Apply**: if the item is not present, or does not hold a draft, does nothing. Otherwise
replaces the title of the draft. **Last-write-wins**, same as `SetTitleOperation`.

### 6.12 ConvertDraftOperation (type 12)

Turns a draft into a reference to an entity: the bug made out of the draft.

| Field     | JSON key    | Type               | Required | Description                             |
|-----------|-------------|--------------------|----------|-----------------------------------------|
| Item      | `item`      | item ID            | yes      | ID of the item                          |
| Namespace | `namespace` | string             | yes      | Namespace, e.g. `bugs`; text, not blank |
| Entity    | `entity`    | entity ID          | yes      | ID of the entity                        |

**Apply**: if the item is not present, or does not hold a draft, does nothing. If another
item present on the board already refers to an entity with this ID, does nothing either.
Otherwise the content of the item becomes the reference to the entity; the item keeps its
ID, its column and its position. Both its ID and the ID of the entity designate it from
then on ([§3.2](#32-items)).

The entity is created by its own operations, under its own reference: a bug is made out of
a draft by the `CreateOperation` of a new bug (see
[bug.md §5.1](bug.md#51-createoperation-type-1)), then by this operation. The two are
fetched separately: a reader may hold the item before the bug, which is then a reference
that does not resolve yet, or the bug before the item.

Of two concurrent conversions of the same draft, the one that sorts first in the total
operation order applies. The entity of the other one stays in the repository, on no board.

Example:
```json
{
  "type": 12,
  "timestamp": 1609473600,
  "nonce": "wIm2hO5fD1eCrNpK9LsWqBAAAAA=",
  "item": "e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6",
  "namespace": "bugs",
  "entity": "a3b4c5d6e7f8a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a1b2c3d4e5f6a7b8"
}
```

### 6.13 SetMetadataOperation (type 13)

See [dag-entity.md §6.6](dag-entity.md#66-setmetadataoperation-generic-pattern) for the
full semantics. The `target` ID refers to any operation in the board's history.

| Field        | JSON key       | Type                   | Required |
|--------------|----------------|------------------------|----------|
| Target       | `target`       | operation ID           | yes      |
| New metadata | `new_metadata` | object (string→string) | no       |


## 7. Snapshot semantics

The snapshot is built by applying operations in the order defined in
[dag-entity.md §9](dag-entity.md#9-operation-ordering). The resulting state is:

| Property    | Type              | Set by                                                                  |
|-------------|-------------------|-------------------------------------------------------------------------|
| ID          | 64-char hex       | Derived from `CreateOperation` ID                                       |
| Title       | string            | Last `SetTitleOperation`; initially from `CreateOperation`              |
| Description | string            | Last `SetDescriptionOperation`; initially from `CreateOperation`        |
| Columns     | ordered list      | The columns present ([§3.1](#31-columns)), by position, then by ID      |
| Items       | ordered list      | The items present ([§3.2](#32-items)), by position, then by ID          |
| Actors      | set of identities | All operation authors                                                   |

The snapshot depends on the operations of the board alone: the entities that its items refer
to are not read to build it.


## 8. Test vectors

Test vectors for this entity are in [`testdata/board.json`](testdata/board.json).
