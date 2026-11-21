ASON Document Format
====================

The ASON Document Format (ASONDOC) is a rich text format based on [ASON](ason.md).

C0 Controls
-----------

In ASONDOC plaintext, the ASON default special values are inherited:

* Boolean values are denoted by [ACK] (true) and [NAK] (false)
* Undefined values are denoted by [ENQ]
* Terminal-safe escape sequences are introduced by [DLE]
* Special processing is indicated by [SYN]
* Paragraph breaks are denoted by [ETB], as per IPTC-7901

ASON excisors are also permitted in the plaintext:

* [CAN] is used to indicate the start of ASON text; any data before it in the current field MUST be discarded.
* [EM] is used to indicate the end of ASON text; any data after it in the current field MUST be discarded.


Htext Records
-------------

* The magic key [SYN] = `^V` MUST be present in the htext and MUST be the first key.
    Its value SHOULD contain a schema identifier, which MAY be the empty string to indicate a schema-free object.
    The value MAY further be terminated by a newline character or character sequence (e.g. [CR][LF] or just [LF]), which SHOULD match the newline format used in the stext if present.
* The key [SYN =] = `^V =` takes a plaintext value which is a unique identifier for the structure.
* The key [SYN -] = `^V -` takes a plaintext value which is a caption or description of the structure.
    The value SHOULD be human-readable.

Several standard schemas are defined, and are indicated by DLE escape sequences (see below).


Structured text schemas
-----------------------

Structured text schemas MAY be explicitly denoted by a header record where the key is `^V` and the value is a metadata string.
The metadata string consists of a [DLE] character followed by a character from the set [ENQ, ACK, NAK, SYN, ETB].

### Quote

The stext of a quote [DLE ENQ] consists of a single opaque value.
Any enclosed ASON structure MUST properly nest but SHOULD NOT be interpreted.

### List

The stext of a list [DLE ACK] contains one or more fields of ASON text, separated by [US].

### Dictionary

The stext of a dictionary [DLE NAK] is represented as

    key1 [ US value1 ] [ RS key2 [ US value2 ] ... ]

It uses [US] to separate the keys from the values, and [RS] to delimit the records.

* Each record SHOULD contain one key and at most one value.
* Keys SHOULD be unique.
* Otherwise, keys are plaintext and values are ASON text.

Note that an htext has the same format as a dictionary's stext, but with additional constraints on the keys and values.

### Database

The stext of a database [DLE SYN] is represented as

    [ SYN US colname1 US colname2 ... ]
    key1 US value(1, col1) US value(1, col2) ... [ RS key2 US value(2, col1) US value(2, col2) ... ]
    [ GS
        [ SYN US colname1a US colname2a ... ]
        key1a US value(1a, key1a) US value(1a, key2a) ... ]
    ...

The group separator [GS] is used to separate individual tables in a database, or equivalently individual sheets in a composite spreadsheet.
If the first row in a table starts with [SYN], it contains column names rather than a record.

* Keys and column names SHOULD be unique.
* Otherwise, keys are plaintext and values are ASON text.
* Databases MAY be ragged-right, however records SHOULD NOT contain more values than the number of column names.

### Array

The stext of an array [DLE ETB] is represented as

    value(1,1) [ US value(1,2) ... ] [ RS value(2,1) [ US value(2,2) ... ] ... ]

It can have up to four dimensions by using all of [FS, GS, RS, US].

* Values are ASON text.
* Arrays MAY be ragged-right.

An array MAY have more than four dimensions by nesting arrays inside array values; 

### Custom or Schema-free Object

The stext of a custom or schema-free object MAY use any combination of [FS, GS, RS, US] to delimit its fields.
A schema-free object with a single field MAY be used to create an ASONDOC magic number where no other structure is required (see below).


Compatibility
=============

Identification
--------------

An ASON document MUST begin with the magic number [SOH SYN US] = `^A ^V ^_` (0x01,0x16,0x1f).
This is the start of an ASON structure with a mandatory header key, so is sufficient to uniquely identify an ASON document.
[SYN] was chosen as the mandatory first key in order to produce a reliable file magic number.

An ASONDOC interpreter MAY search for this magic number at non-initial positions in a stream or file if the application allows it.
For example, an ASON-aware script interpreter MAY ignore leading lines of the form `#!/path/to/executable\n`.
If ASON data starts at a non-initial file position it MUST either:

* be preceded a CAN character, i.e. [CAN SOH SYN US] = `^X ^A ^V ^_`, and the interpretation of any preceding bytes is application-dependent.
* be preceded only by an initial byte-order mark (BOM) (see "USON" below).

The magic number SHOULD be followed by a newline sequence, then either [RS] or [STX].

Graceful degradation
--------------------

ASCII has been a longstanding source of incompatibility:

* Linebreak sequences are either [CR, LF, CR LF] depending on context.
* Extended-ASCII documents may also use [NEL, LS, PS].
* Backspace may be used to indicate overstrike.

ASON has the potential to add further formatting chaos:

* Parsers may expect to process newline-terminated records, and may run out of buffer if newlines are not encountered frequently in tabular input.
* Legacy applications may not expect to process ASON, which may cause issues in parsing and display.
* Text viewers may not understand the ASON delimiters, leading to formatting issues.

We therefore allow ASON to have optional embedded compatibility characters that format the output when processed in legacy applications, but which ASON-aware applications MUST ignore.

* If a field contains the character [CAN], any ASON plaintext characters before it MUST be discarded.
* If a field contains the character [EM], any ASON plaintext characters after it MUST be discarded.
* A field MUST NOT contain more than one of each of the excisor characters, unless they are contained within a nested structure.

    structure [ plaintext CAN ] [ encoding ] data [ EM plaintext ] structure

For example, the plaintext preceding [CAN] might contain whitespace indentation, and the plaintext following [EM] might contain a comment and/or line break.
This allows a non-ASON editor or terminal to display structured data in human-readable form, while the ASON structure encodes the same information in machine-readable form.

An excisor-only byte string such as [CAN, EM, CAN EM] can be used to explicitly denote an empty string value where it would otherwise be ambiguous.


Terminal-Safe Escape Sequences
==============================

DLE sequences were defined in ECMA-24 and ECMA-37, however these were rarely used and are now considered obsolete.
We have repurposed a subset of DLE sequences that are syntactically compatible with ECMA-37, but with novel semantics.

As per ECMA-37, [DLE] is used to escape the normal meaning of the following character, in particular [ENQ, ACK, DLE, NAK, SYN, ETB].
An application MAY use these sequences to denote structure metadata using terminal-safe C0 codes:

```
[DLE ENQ] `^P ^E` 0x10,0x05 (quote structure)
[DLE ACK] `^P ^F` 0x10,0x06 (list structure)
[DLE DLE] `^P ^P` 0x10,0x10 (=[DLE])
[DLE NAK] `^P ^U` 0x10,0x15 (dictionary structure)
[DLE SYN] `^P ^V` 0x10,0x16 (database structure)
[DLE ETB] `^P ^W` 0x10,0x17 (array structure)
```

The following valid DLE sequences are forbidden in ASON (even at the application layer):

```
[DLE SOH] `^P ^A` 0x10,0x01
[DLE STX] `^P ^B` 0x10,0x02
[DLE ETX] `^P ^C` 0x10,0x03
[DLE EOT] `^P ^D` 0x10,0x04
```

Other DLE sequences containing ASCII graphical codepoints are permitted by ECMA-37, and are safe to use in ASON plaintext, but these have been left undefined to avoid line noise.
