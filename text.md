# Specification: text

This document specifies which strings are text: the strings shown to people, such as the
title of a bug, a comment, the name of an identity or the description of a label. The
specifications of the entities and of identities refer to it for each such string.


## 1. Overview

A string that should be text but isn't makes its operation or version invalid, and every
client must reach the same verdict: one that accepts what another rejects ends up with a
different history. The rules below are therefore exact, and stated in code points and
Unicode properties.

They keep out the characters that break the display of a string or hide part of it: control
characters, the characters that reverse the direction of the text around them, and the
noncharacters, which Unicode reserves for internal use.


## 2. Text

*Text* is a string holding none of these characters:

- the control characters, U+0000 to U+001F and U+007F to U+009F, among them the line breaks
  and the tab;
- the bidirectional embeddings and overrides, U+202A to U+202E;
- the noncharacters: U+FDD0 to U+FDEF, and the last two code points of every plane (U+FFFE,
  U+FFFF, U+1FFFE, U+1FFFF, and so on up to U+10FFFF).

Text is a single line. The bidirectional marks and isolates, such as U+200F or U+2066, are
allowed, as text written from right to left needs them.


## 3. Multi-line text

*Multi-line text* follows the same rules as text, except that it may also hold the tab
(U+0009), the line feed (U+000A) and the carriage return (U+000D).


## 4. Blank text

Text or multi-line text is *blank* when each of its characters is a separator, a control
character, a format character or a private-use character (Unicode general categories Z, Cc,
Cf and Co, as of Unicode 17.0.0). It is then empty, or holds only characters such as spaces,
line breaks, U+200B ZERO WIDTH SPACE or private-use characters. A code point that Unicode
17.0.0 leaves unassigned belongs to none of these categories, so text holding one is not
blank.

A field that must not be blank says so. Otherwise it may be blank, and in particular empty.


## 5. Length

The length of a string is its number of code points, not of bytes: `é` written as U+00E9
has a length of 1, and as `e` followed by U+0301 a length of 2.


## 6. Test vectors

Test vectors are in [`testdata/text.json`](testdata/text.json).
