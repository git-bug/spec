# Specification: JSON and text

This document specifies the rules that every format shares: how to read a JSON document,
which strings are text, and which are URLs a client may fetch. The specifications of the entities and of identities refer to
it.


## 1. Overview

Every client must reach the same verdict on a pack, an operation or a version: one that
accepts what another rejects ends up with a different history. The rules below are
therefore exact.

The JSON rules settle the points where JSON libraries disagree: invalid UTF-8, a byte
order mark, lone surrogates, repeated member names, deep nesting, the form and range of
integers, and members that are `null`.

*Text* is a string shown to people, such as the title of a bug, a comment, the name of an
identity or the description of a label. The text rules, stated in code points, keep out the
characters that break the display of a string or hide part of it: control characters, line
separators in a single line, the characters that change the direction of the text displayed
after them, and the noncharacters, which Unicode reserves for internal use.


## 2. JSON documents

The `ops` blob of an entity and the `version` blob of an identity are *JSON documents*: JSON
([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)) encoded in UTF-8. A reader **must**
reject a document that:

- is not valid UTF-8;
- holds an escaped surrogate, `\uD800` to `\uDFFF`, other than a high surrogate followed by
  a low one;
- starts with a byte order mark, U+FEFF;
- holds an object with two members of the same name;
- nests objects and arrays more than 64 deep, the outermost value counting as 1.

Member names match exactly, case included: in a bug operation, `Title` is an unknown member.
Many JSON decoders replace invalid UTF-8 or a lone surrogate with U+FFFD, skip a byte order
mark, keep the last of two members of the same name, or match names regardless of case, so
a reader checks these rules on the blob itself. Each decoder also stops at a depth of its
own, and 64 lies within the limit of the common ones.


## 3. Members, maps and null

An object in a JSON document is of one of two kinds:

- an object of *members*, whose names the format defines;
- a *map*, whose keys are data.

Each format names its maps, and defines what each member means when it is absent: a default
value, a value left unchanged, or an invalid operation or version when the member is
required.

A reader reads `null` as follows:

- a member that is `null` is absent;
- a value that is `null` in a map is the empty value of its type, the empty string or 0, and
  its key stays;
- an array holding `null` makes its operation or version invalid.

A writer never writes `null`: it leaves the member out, or writes the empty value in a map.
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


## 5. Text

*Text* is a string holding none of these characters:

- the control characters, U+0000 to U+001F and U+007F to U+009F, among them the line breaks
  and the tab;
- U+2028 LINE SEPARATOR and U+2029 PARAGRAPH SEPARATOR;
- the bidirectional embeddings and overrides, U+202A to U+202E;
- the surrogates, U+D800 to U+DFFF;
- the noncharacters: U+FDD0 to U+FDEF, and the last two code points of every plane (U+FFFE,
  U+FFFF, U+1FFFE, U+1FFFF, and so on up to U+10FFFF).

Text is a single line. The bidirectional marks, such as U+200F, are allowed, as text written
from right to left needs them, and so are the isolates when they pair up.

The isolates U+2066, U+2067 and U+2068 each open a span laid out on its own, and U+2069
closes the last one still open. In text, they *pair up*: reading the string from its start,
each U+2069 closes an isolate that is open, and no isolate is open at the end. An isolate
left open would carry on over what a client displays after the string, such as the author
next to the title of a bug.


## 6. Multi-line text

*Multi-line text* follows the same rules as text, except that it may also hold the tab
(U+0009), the line feed (U+000A), the carriage return (U+000D), U+2028 LINE SEPARATOR and
U+2029 PARAGRAPH SEPARATOR. Its isolates pair up within each line, a line ending at a line
feed, a carriage return or U+2029 PARAGRAPH SEPARATOR, as in the Unicode Bidirectional
Algorithm.


## 7. Blank text

Text or multi-line text is *blank* when each of its code points is in the list below: the
spaces and line breaks, the control and format characters, the private-use characters and
the characters that display as nothing. Empty text is blank.

| Code points                          | Characters                                                        |
|--------------------------------------|-------------------------------------------------------------------|
| U+0000–U+0020                        | Control characters, space                                         |
| U+007F–U+00A0                        | Control characters, no-break space                                |
| U+00AD                               | Soft hyphen                                                       |
| U+034F                               | Combining grapheme joiner                                         |
| U+0600–U+0605                        | Arabic number signs                                               |
| U+061C                               | Arabic letter mark                                                |
| U+06DD                               | Arabic end of ayah                                                |
| U+070F                               | Syriac abbreviation mark                                          |
| U+0890–U+0891                        | Arabic pound and piastre marks above                              |
| U+08E2                               | Arabic disputed end of ayah                                       |
| U+115F–U+1160                        | Hangul fillers                                                    |
| U+1680                               | Ogham space mark                                                  |
| U+17B4–U+17B5                        | Khmer inherent vowels                                             |
| U+180B–U+180F                        | Mongolian variation selectors and vowel separator                 |
| U+2000–U+200F                        | Spaces, zero width characters, direction marks                    |
| U+2028–U+202F                        | Line and paragraph separators, embeddings and overrides, space    |
| U+205F–U+206F                        | Mathematical space, word joiner, invisible operators, isolates    |
| U+2800                               | Braille pattern blank                                             |
| U+3000                               | Ideographic space                                                 |
| U+3164                               | Hangul filler                                                     |
| U+E000–U+F8FF                        | Private use                                                       |
| U+FE00–U+FE0F                        | Variation selectors                                               |
| U+FEFF                               | Zero width no-break space                                         |
| U+FFA0                               | Halfwidth Hangul filler                                           |
| U+FFF0–U+FFFB                        | Reserved, interlinear annotation characters                       |
| U+110BD, U+110CD                     | Kaithi number signs                                               |
| U+13430–U+1343F                      | Egyptian hieroglyph format controls                               |
| U+1BCA0–U+1BCA3                      | Shorthand format controls                                         |
| U+1D173–U+1D17A                      | Musical symbol format characters                                  |
| U+E0000–U+E0FFF                      | Tags, variation selectors                                         |
| U+F0000–U+FFFFD, U+100000–U+10FFFD   | Private use                                                       |

The list gathers the Unicode 17.0.0 general categories Z, Cc, Cf and Co, the default
ignorable code points and U+2800 BRAILLE PATTERN BLANK. It is fixed: a reader uses it as it
stands, whatever version of Unicode it knows.

A field that must not be blank says so. Otherwise it may be blank, and in particular empty.


## 8. Length

The length of a string is its number of code points, not of bytes: `é` written as U+00E9
has a length of 1, and as `e` followed by U+0301 a length of 2.


## 9. HTTPS URLs

An *HTTPS URL* is a URL that a client may fetch on its own, without the user following it,
such as the picture of an identity. It is a string of at most 2048 characters made of, in
order:

1. `https://`;
2. a host: one or more ASCII letters, digits, `-` and `.`, optionally followed by `:` and
   one or more digits;
3. optionally, a `/`, `?` or `#` followed by any number of ASCII letters, digits and
   characters among `-._~:/?#[]@!$&'()*+,;=%`, each `%` followed by two hexadecimal digits.

The rule is exact, as URL parsers differ from one language to another, and it holds what a
client fetches to an HTTPS host, named in full.


## 10. Test vectors

Test vectors for text are in [`testdata/text.json`](testdata/text.json), and for HTTPS URLs
in [`testdata/url.json`](testdata/url.json). Those for the JSON rules are among the vectors
of each format: the `ops` blobs of [`testdata/dag-entity.json`](testdata/dag-entity.json),
and the operations of [`testdata/bug.json`](testdata/bug.json) and
[`testdata/config.json`](testdata/config.json).
