# Specification: positions

This document specifies the positions that order the elements of a list in the
[DAG entity format](dag-entity.md): the entries of an ordered list of the
[config](config.md#74-ordered-lists), and the columns and items of a [board](board.md).


## 1. Overview

Each element of a list carries a **position**, and the list is read in the order of the
positions. Inserting or moving an element sets the position of that element alone: there is
always a position between two others, so no other element is renumbered. Two people can
therefore insert or move elements of the same list at the same time, and both changes take
effect.

A position is a value of the element like any other. Which operation sets it, and how
elements with the same position are ordered, is defined by the specification of each list.


## 2. Format

A position is a fractional index, in the format described by David Greenspan in
[Implementing Fractional Indexing](https://observablehq.com/@dgreensp/implementing-fractional-indexing).
It is a string of the digits `0`–`9`, `A`–`Z` and `a`–`z`, whose order is also the order of
their bytes, made of two parts:

1. An integer part, whose first character, a letter, gives its length, that character
   included: 2 for `a`, 3 for `b`, up to 27 for `z`, and 2 for `Z`, 3 for `Y`, up to 27 for
   `A`. The lowercase letters start the integers from `a0` upward, the uppercase ones those
   below it: `Zy`, `Zz`, `a0`, `a1`, …, `az`, `b00`, `b01`.
2. A fractional part: the rest of the string, possibly empty, not ending with `0`.

`A` followed by 26 `0`, with nothing after it, is not a position.


## 3. Order

Positions are compared byte by byte: `Zz`, `a0`, `a0G`, `a0V`, `a1`, `az`, `b00` are in
order.

There is always a position before another, after another, and between two different ones:
`Zz` lies before `a0`, `a1` after `a0`, `a0V` between `a0` and `a1`, and `a0G` between `a0`
and `a0V`.

Two elements of a list may have the same position, for example when two people insert at
the same place at the same time. Each list defines the order of such elements, so that every
reader lists them the same way.


## 4. Choosing a position

A writer inserts or moves an element by setting its position, and **must** set it between
the positions of its new neighbours, or before the first or after the last element, without
changing any other element.

When no position lies between the new neighbours, because they have the same position, the
writer **must** first move enough of the elements involved to new positions, in the same
pack and keeping their order, so that one does.

Any position that satisfies this is valid, and readers treat them all alike. Which one a
writer picks matters over time:

- At either end of a list, stepping the integer part (`a1`, `a2`, …) keeps the positions
  short when elements are added one after another.
- Between two elements, the midpoint that Greenspan describes keeps the positions short
  when elements are inserted again and again at the same place.
- Two writers that insert at the same place at the same time, and both take the midpoint,
  give their elements the same position. Where this is likely, as on a board edited by
  several people, a writer **should** instead pick at random among the positions between
  the two neighbours.
- A writer that inserts several elements at the same place **should** give them positions
  close to each other, around a single point picked at random between the neighbours.
  Elements that someone else inserts at the same place, at the same time, then fall before
  or after the group rather than inside it.


## 5. Test vectors

Test vectors for positions are in [`testdata/positions.json`](testdata/positions.json).
