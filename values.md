# Specification: JSON documents

This document specifies the rules that every format shares on how to read a JSON document.
The specifications of the entities and of identities refer to it.


## 1. Overview

Every client must reach the same verdict on a pack, an operation or a version: one that
accepts what another rejects ends up with a different history. The rules below are
therefore exact.

The JSON rules settle the points where JSON libraries disagree: invalid UTF-8, a byte
order mark, lone surrogates, repeated member names, the form and range of integers, and
members that are `null`.


## 2. JSON documents

The `ops` blob of an entity and the `version` blob of an identity are *JSON documents*: JSON
([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)) encoded in UTF-8. A reader **must**
reject a document that:

- is not valid UTF-8;
- holds an escaped surrogate, `\uD800` to `\uDFFF`, other than a high surrogate followed by
  a low one;
- starts with a byte order mark, U+FEFF;
- holds an object with two members of the same name.

Member names match exactly, case included: in a bug operation, `Title` is an unknown member.
Many JSON decoders replace invalid UTF-8 or a lone surrogate with U+FFFD, skip a byte order
mark, keep the last of two members of the same name, or match names regardless of case, so
a reader checks these rules on the blob itself.


## 3. Members, maps and null

An object in a JSON document is of one of two kinds:

- an object of *members*, whose names the format defines;
- a *map*, whose keys are data.

Each format names its maps, and defines what each member means when it is absent: a default
value, a value left unchanged, or an invalid operation or version when the member is
required.

A reader reads `null` as follows:

- a member that is `null` is absent;
- a map or an array holding `null` makes its operation or version invalid.

A writer never writes `null`: it leaves the member out.
`null` thus carries no meaning of its own.

A reader applies these rules to what it decodes. An operation of an unknown type is kept
verbatim, without being decoded
([dag-entity.md §6.4](dag-entity.md#64-unknown-operation-types)).


## 4. Integers

An *integer* is written as digits with an optional leading `-`, without a fraction or an
exponent, and lies between −(2⁵³ − 1) and 2⁵³ − 1. Every JSON implementation reads such a
number exactly. Beyond that range, many round it, and decoders disagree on `1.0` or `1e3`:
some read an integer, others reject it.

A member or map value that a format defines as an integer holds an integer, or its
operation or version is invalid: a `timestamp` of `1609459200.0`, `1.6e9` or
`9007199254740992` makes its operation invalid.


## 5. Test vectors

Test vectors for the JSON rules are among the vectors of each format: the `ops` blobs of
[`testdata/dag-entity.json`](testdata/dag-entity.json), and the operations of
[`testdata/bug.json`](testdata/bug.json) and [`testdata/config.json`](testdata/config.json).
