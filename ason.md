ASCII structured object notation
================================

ASCII structured object notation (ASON) is a non-binary serialised data format with the following unique properties:

* It is human-readable and -writable (with suitable editor support).
* It can be handled safely by non-ASON-aware text applications, including terminal emulators.
* It minimizes the need for escape sequences and quoting.
* Plaintext data can be trivially embedded inside an ASON document.
* Any ASON document can be trivially embedded inside another ASON document.

ASON does this by using the C0 information separator characters to impose a hierarchical structure on textual data, and the C0 transmission control characters to convey metatextual information.

ASON is a pure 7-bit ASCII encoding, and is therefore automatically compatible with 8-bit extended-ASCII encodings such as ISO-2022 and UTF-8.
It can also contain arbitrary data via binary-to-text encoding (e.g. base64, quoted-printable).

ASON is to ASCII what REST is to HTTP.

References
----------

* ASCII is defined in ISO-646/ECMA-6: https://www.ecma-international.org/publications-and-standards/standards/ecma-6/
* C0 control codes are defined in ISO-6429/ECMA-48: https://www.ecma-international.org/publications-and-standards/standards/ecma-48/
* Transmission control characters TC1-TC10 are defined in ISO-1745/ECMA-16 : https://www.ecma-international.org/publications-and-standards/standards/ecma-16/
    * DLE sequences are defined in ECMA-37: https://www.ecma-international.org/publications-and-standards/standards/ecma-37/
* Base64 and quoted-printable encodings are defined in RFC-2045: https://datatracker.ietf.org/doc/html/rfc2045
* A similar application of C0 control characters can be found in IPTC-7901: https://www.iptc.org/std/IPTC7901/1.0/specification/7901V5.pdf
* A (joke?) encoding similar to ASON was previously proposed by Terence Eden: https://shkspr.mobi/blog/2017/03/kyli-because-it-is-superior-to-json/

Primitive encoding
------------------

ASON text may contain any byte that is not explicitly forbidden.
The seven forbidden characters have common special meanings when text is stored in memory or a file, or transmitted over a data link, and are therefore excluded on safety grounds.

* "ASON forbidden" - 0x00 [NUL], 0x04 [EOT], 0x11..0x14 [DC1-4], or 0x1a [SUB]

A further seven bytes are reserved to the ASON structure layer and have the same meaning regardless of context.

* "ASON structure" - 0x01..0x03 [SOH, STX, ETX] and 0x0c..0x0f [FS, GS, RS, US]

Two bytes indicate that leading or trailing data should be discarded by an ASON-aware application (see below).

* "ASON excisor" - 0x18, 0x19 [CAN, EM]

"ASON plaintext" is any other sequence of one or more bytes.
These are further classified:

* "ASON special value" - 0x05 [ENQ], 0x06 [ACK], 0x10 [DLE], or 0x15..0x17 [NAK, SYN, ETB] (see below)
* ASCII 0x07 [BEL] is treated as opaque
* "ASCII format effector" - 0x08..0x0d [BS, HT, LF, VT, FF, CR]
* ASCII 0x0e, 0x0f, 0x1b [SI, SO, ESC] are treated as per ISO-2022; any code-shift or escape sequences MUST be properly reverted or terminated.
* "ASCII graphical" - 0x20..0x7f, which take their usual ASCII meanings
* "extended-ASCII" - 0x80..0xff (encoding-dependent, treated as opaque)

### Special values

In ASON plaintext, the following standard meanings are defined:

* Boolean values are denoted by [ACK] (true) and [NAK] (false)
* Undefined values are denoted by [ENQ]
* Paragraph breaks are denoted by [ETB], as per IPTC-7901

### Excisors

* [CAN] is used to indicate the start of ASON text; any data before it in the current field MUST be discarded.
* [EM] is used to indicate the end of ASON text; any data after it in the current field MUST be discarded.

[CAN, EM] are also used to indicate the start and end of an ASON document embedded within a larger file.
Excisor characters may also be used to explicitly disambiguate a zero-length string (see below).

Structure encoding
------------------

A structure consists of a header text (htext) and an optional structured text (stext), delimited by [SOH, STX, ETX] = `^A ^B ^C` in the form:

    SOH htext [ STX stext ] ETX

The htext and stext consist of one or more fields, delimited by the information separators [FS, GS, RS, US] = `^\ ^] ^^ ^_` .

### htext

The htext is a sequence of one or more key-value records of the form:
    
    key US value [ RS key US value ... ]
    
where the field delimiters are [RS, US] = `^^ ^_`.

* The magic key [SYN] = `^V` MUST be present in the htext and MUST be the first key.
    It takes a value that indicates the format of the structure (see below).
* The key [SYN =] = `^V =` takes a plaintext value which is a unique identifier for the structure.
* The key [SYN -] = `^V -` takes a plaintext value which is a caption or description of the structure.
    The value SHOULD be human-readable.
* The key [SYN !] = `^V !` takes a plaintext value which indicates the schema for an object structure and/or any application-defined field encodings used in the structure (see below).
    The value SHOULD be globally unique, and reverse-domain format is RECOMMENDED.
    An htext schema applies only to the current structure, and is not inherited by any structures embedded in its stext.

Keys starting with [SYN] are reserved to ASON.
Applications MAY define their own htext keys, but they MUST NOT contain C0 characters.
Htext values MUST NOT contain C0 characters.

### stext

The nature of the stext fields differs between structured text formats (see below).

* If the stext is empty, the structure contains a single empty field.
* If the stext is missing, the structure contains no data.

### Nesting

* A structure MAY be nested within another structure's stext.
* A structure MUST NOT be nested within an htext.
* All ASON texts used in the stext MUST be well-formed and properly nested.
* Any ISO-2022 locking-shift or escape sequences MUST be properly reverted or terminated at the end of each field, and before each embedded structure.

Any data that violates the above rules MUST be binary-to-text encoded (see below).

Note that child structures MUST NOT inherit any metadata (including ISO-2022 encodings) from their parent.
This ensures that ASON documents can be trivially embedded within each other without altering their semantics.

Structured text formats
-----------------------

Structured text formats are denoted by a key-value pair in the first record of the htext, where the key is [SYN] and the value is a metadata string.
The metadata string consists of a [DLE] character followed by a character from the set [ENQ, ACK, DLE, NAK, SYN, ETB].

Fields may contain the empty string, the `undefined` character [ENQ], or be missing entirely.
Note that care MUST be taken to denote empty strings in an unambiguous manner.
To prevent zero-length values from being misinterpreted as multi-character data separators (see below), zero-length fields SHOULD contain either:

* at least one excisor character to delimit the empty string
* an [ENQ] character to represent "undefined"

An application MAY distinguish between zero-length and undefined values.

### Quote

The stext of a quote [SOH SYN US DLE ENQ] = `^A ^V ^_ ^P ^E` is opaque.
Any enclosed ASON structure MUST properly nest but SHOULD NOT be interpreted.

### List

The stext of a list [SOH SYN US DLE ACK] = `^A ^V ^_ ^P ^F` contains one or more fields of ASON text, separated by [US].
A list with a single field MAY be used to create a file magic number where no other structure is required (see below).

### Object

The stext of an object [SOH SYN US DLE DLE] = `^A ^V ^_ ^P ^P` is application-defined.
It MAY use any combination of [FS, GS, RS, US] to delimit its fields.

It is RECOMMENDED that the schema of an object be indicated in the htext using the key `^V !` (see above).
This schema SHOULD define any application-defined field encodings that appear directly in the object.

### Dictionary

The stext of a dictionary [SOH SYN US DLE NAK] = `^A ^V ^_ ^P ^U` is represented as

    key1 [ US value1 ] [ RS key2 [ US value2 ] ... ]

It uses [US] to separate the keys from the values, and [RS] to delimit the records.

* Each record SHOULD contain one key and at most one value.
* Keys SHOULD be unique.
* Otherwise, keys are plaintext and values are ASON text.

Note that an htext has the same format as a dictionary's stext, but with additional constraints on the keys and values.

### Table

The stext of a table [SOH SYN US DLE SYN] = `^A ^V ^_ ^P ^V` is represented as
    
    key1 US key2 ... GS value(1,key1) US value(1,key2) ... [ RS value(2,key1) US value(2,key2) ... ]

The group separator [GS] is used to separate the first row, containing the column names (keys), from the rows (records) containing the values.

* Keys SHOULD be unique.
* Otherwise, keys are plaintext and values are ASON text.
* Tables MAY be ragged-right, however records SHOULD NOT contain more values than the number of columns.

### Array

The stext of an array [SOH SYN US [US ...] DLE ETB] = `^A ^V ^_ [^_ ...] ^P ^W` is represented as

    value(1,1) [ US value(1,2) ... ] [ RS value(2,1) [ US value(2,2) ... ] ... ]

It can have up to four dimensions by using all of [FS, GS, RS, US].

* Values are ASON text.
* Arrays MAY be ragged-right.

An array MAY have more than four dimensions by using multi-character sequences as separators:

* up to 16 by using the big-endian two-character sequences [FS FS, FS GS, FS RS, ... US GS, US RS, US US]
* up to 64 by using three-character sequences [FS FS FS, FS FS GS, ... US US GS, US US RS, US US US]
* etc.

Field encoding
--------------

Data that is not valid ASON text MUST be binary-to-text encoded.
If a field's content begins (after any [CAN] excisor) with a [DLE] sequence, it indicates the encoding used for its contents:

* [DLE ENQ] = `^P ^E` quoted-printable
* [DLE ACK] = `^P ^F` base64
* [DLE DLE] = `^P ^P` application-defined
* [DLE NAK] = `^P ^U` none (default)

The `base64` and `quoted-printable` encodings are as defined in RFC 2045.
Applications SHOULD support `base64`, and MAY support `quoted-printable` and/or `application-defined`.
If the data itself begins with a [DLE] character but is not going to be encoded, it MUST be prefixed by an explicit `none` encoding indicator.


Compatibility
=============

Identification
--------------

An ASON document MUST begin with the magic number [SOH SYN US] = `^A ^V ^_` (0x01,0x16,0x1f).
This is the start of an ASON structure, and is sufficient to uniquely identify an ASON document.
[SYN] was chosen as the mandatory first key in order to produce a reliable file magic number.

An ASON interpreter MAY search for this magic number at non-initial positions in a stream or file if the application allows it.
For example, an ASON-aware script interpreter MAY ignore leading lines of the form `#!/path/to/executable\n`.
If ASON data starts at a non-initial file position it MUST either:

* be preceded a CAN character, i.e. [CAN SOH SYN US] = `^X ^A ^V ^_`, and the interpretation of any preceding bytes is application-dependent.
* be preceded only by an initial byte-order mark (BOM) (see "USON" below).

Concatenation of documents
--------------------------

An ASON file or data stream MAY contain one or more concatenated ASON structures at the top level; these MUST be interpreted as independent documents.
An [EM] character immediately following the outermost [ETX] character of an ASON document disables ASON interpretation for the rest of the file or data stream.
The meaning of any subsequent bytes is application-dependent.

An application SHOULD append [EM] to the end of the final (or only) ASON document in a file or data stream, and MUST do so if there is any subsequent non-ASON data.
An application MAY require a trailing [EM] to verify non-truncation of input data.
When concatenating multiple ASON documents, any intervening non-ASON data, including excisors, MUST be removed.

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

ASON-aware text display/edit
----------------------------

An ASON-aware text display system SHOULD indicate ASON structure in a distinctive visual manner.
One simple method would be as follows:

* ASON structures are displayed outlined or distinctly coloured.
* [RS, US] delimited fields are arranged in 1-D or 2-D tables or arrays.
* [FS, GS] delimiters are indicated by hlines and/or distinct colours.
* Boolean values are shown using checkbox glyphs or similar, either inverse video or distinctly coloured.

An ASON-aware text input system SHOULD allow the user to enter ASON structure info in a standard manner:

* A typed Ctrl-A SHOULD indicate a new header and a typed Ctrl-B SHOULD end the header and start the text.
    * The editor SHOULD autocomplete the magic key and offer the user a menu of structure types.
    * The editor SHOULD autocomplete the trailing [ETX] so that the user does not have to type Ctrl-C.
* [FS, GS, RS, US] SHOULD be entered using Ctrl-4 to Ctrl-7

An ASON-aware IDE MAY also conform to the following:

* Characters or character sequences with special meaning in a particular application context MAY be mapped onto one of the ASON control characters, or an ASON structure.
* This mapping MAY be performed in real time if the application allows it.


Data representation
===================

At the data layer ASON is encoded as a sequence of bytes.
Some of these are interpreted as ASCII control characters with special meaning to the structure and plaintext layers of ASON.
Any byte in the range 0x20..0xff is treated as plaintext by ASON.

C0 control chars reserved to the structure layer
------------------------------------------------

These bytes are reserved to the structure of ASON.
They MUST NOT appear unless as part of a well-formed and properly nested ASON structure.

```
[SOH] `^A` 0x01 (start header)
[STX] `^B` 0x02 (start text)
[ETX] `^C` 0x03 (end text)
[FS]  `^\` 0x1c (file separator)
[GS]  `^]` 0x1d (group separator)
[RS]  `^^` 0x1e (record separator)
[US]  `^_` 0x1f (unit separator)
```

C0 control characters used for compatibility with non-ASON applications
-----------------------------------------------------------------------

```
[CAN] `^X` 0x18 (start of data / ignore preceding data)
[EM]  `^Y` 0x19 (end of data / ignore trailing data)
```

C0 control characters with special meanings in the plaintext layer
------------------------------------------------------------------

The following control characters are understood by ASON at the metadata layer, and MAY be used at the application layer:

```
[ENQ] `^E` 0x05 (undefined)
[ACK] `^F` 0x06 (true)
[DLE] `^P` 0x10 (ECMA-37 escape sequence, see below)
[NAK] `^U` 0x15 (false)
[SYN] `^V` 0x16 (magic)
[ETB] `^W` 0x17 (paragraph)
```

### DLE escape sequences

DLE sequences were defined in ECMA-24 and ECMA-37, however these were rarely used and are now considered obsolete.
We have repurposed a subset of DLE sequences that are syntactically compatible with ECMA-37, but with novel semantics.

As per ECMA-37, [DLE] is used to escape the normal meanings of [ENQ, ACK, DLE, NAK, SYN, ETB].
ASON uses these sequences to denote structure and binary-to-text encoding metadata using terminal-safe C0 codes:

```
[DLE ENQ] `^P ^E` 0x10,0x05 (quote structure, quoted-printable encoding)
[DLE ACK] `^P ^F` 0x10,0x06 (list structure, base64 encoding)
[DLE DLE] `^P ^P` 0x10,0x10 (object structure, application-defined encoding)
[DLE NAK] `^P ^U` 0x10,0x15 (dictionary structure, no encoding)
[DLE SYN] `^P ^V` 0x10,0x16 (table structure, RESERVED encoding)
[DLE ETB] `^P ^W` 0x10,0x17 (array structure, RESERVED encoding)
```

The following valid DLE sequences are forbidden in ASON (even at the application layer):

```
[DLE SOH] `^P ^A` 0x10,0x01
[DLE STX] `^P ^B` 0x10,0x02
[DLE ETX] `^P ^C` 0x10,0x03
[DLE EOT] `^P ^D` 0x10,0x04
```

Other DLE sequences containing ASCII graphical codepoints are permitted by ECMA-37, and are safe to use in ASON plaintext, but these have been left undefined to avoid line noise.

C0 control characters with standard meanings in the plaintext layer
-------------------------------------------------------------------

The BEL, format effector and charset shift characters MAY appear in the plaintext layer of ASON, and have their usual meanings.
If any locking-shift or escape sequences are used, they SHOULD be well-terminated and terminal-safe.

```
[BEL] `^G` 0x07 (opaque)
[BS]  `^H` 0x08 (format effector)
[HT]  `^I` 0x09 (format effector)
[LF]  `^J` 0x0a (format effector)
[VT]  `^K` 0x0b (format effector)
[FF]  `^L` 0x0c (format effector)
[CR]  `^M` 0x0d (format effector)
[SI]  `^N` 0x0e (ISO-2022)
[SO]  `^O` 0x0f (ISO-2022)
...
[ESC] `^[` 0x1b (ISO-2022)
```

C0 control characters forbidden
-------------------------------

These bytes MUST NOT appear anywhere in an ASON document.

```
[NUL] `^@` 0x00
[EOT] `^D` 0x04
[DC1] `^Q` 0x11
[DC2] `^R` 0x12
[DC3] `^S` 0x13
[DC4] `^T` 0x14
[SUB] `^Z` 0x1a
```


Derived formats
===============

USON
----

ASON is an extended-ASCII format, and is agnostic about the particular extended-ASCII encoding used in the plaintext layer.
An application MAY transcode this to UTF-16 or UTF-32 for internal use, in which case all C0 control codes in the ASON structure MUST also be mapped to the corresponding UTF-16/32 codepoints, and the resulting format is called Unicode Structured Object Notation (USON).

If USON is written to persistent storage or passed between applications, then a byte-order mark (BOM) SHOULD be prepended to the beginning of the document, i.e. [BOM SOH SYN US ...].
This produces one of four magic numbers:

* 0xff 0xfe 0x01 0x00 0x16 0x00 0x1f 0x00 (UTF-16 little-endian)
* 0xfe 0xff 0x00 0x01 0x00 0x16 0x00 0x1f (UTF-16 big-endian)
* 0xff 0xfe 0x00 0x00 0x01 0x00 0x00 0x00 0x16 0x00 0x00 0x00 0x1f 0x00 0x00 0x00 (UTF-32 little-endian)
* 0x00 0x00 0xfe 0xff 0x00 0x00 0x00 0x01 0x00 0x00 0x00 0x16 0x00 0x00 0x00 0x1f (UTF-32 big-endian)

Unicode transcoding roundtrips SHOULD be idempotent, therefore an ASON application SHOULD tolerate a UTF-8 BOM being prepended to UTF-8 encoded ASON (see "Magic Number" above).

ESON
----

It is also possible to define EBCDIC Structured Obect Notation by extension.
While ESON is not expected to be used natively by any application, it is defined so that a roundtrip conversion of ASCII->EBCDIC->ASCII remains well-behaved.

All of the C0 controls with meaning at the ASON layer have equivalents in EBCDIC, many of them with the same binary representation (indicated by **).

Forbidden characters:

```
[NUL] ** 0x00
[DC1] ** 0x11
[DC2] ** 0x12
[DC3] ** 0x13
[DC4] ** 0x14
[EOT]    0x37 (ASCII 0x04)
[SUB]    0x3f (ASCII 0x1a)
```

Structure controls:

```
[SOH] ** 0x01 (start header)
[STX] ** 0x02 (start text)
[ETX] ** 0x03 (end text)
[FS]  ** 0x1c (file separator)
[GS]  ** 0x1d (group separator)
[RS]  ** 0x1e (record separator)
[US]  ** 0x1f (unit separator)
```

Excisors:

```
[CAN] ** 0x18 (start of data / ignore preceding data)
[EM]  ** 0x19 (end of data / ignore trailing data)
```

Special values:

```
[ENQ]    0x2d (undefined)   (ASCII 0x05)
[ACK]    0x2e (true)        (ASCII 0x06)
[DLE] ** 0x10 (escape)
[NAK]    0x3d (false)       (ASCII 0x15)
[SYN]    0x32 (magic)       (ASCII 0x16)
[ETB]    0x26 (paragraph)   (ASCII 0x17)
```

Plaintext C0 controls:

```
[BEL]    0x2f (opaque)      (ASCII 0x07)
[BS]     0x16 (format)      (ASCII 0x08)
[HT]     0x05 (format)      (ASCII 0x09)
[LF]     0x25 (format)      (ASCII 0x0a)
[VT]  ** 0x0b (format)
[FF]  ** 0x0c (format)
[CR]  ** 0x0d (format)
[SI]  ** 0x0e (ISO-2022)
[SO]  ** 0x0f (ISO-2022)
[ESC]    0x27 (ISO-2022)    (ASCII 0x1b)
```

EBCDIC code points 0x15 [NEL] and 0x40..0xff are treated as plaintext.
The meaning of any other EBCDIC code point not listed above is undefined.

The representation of [SYN] differs between ASCII (0x16) and EBCDIC (0x32), so ESON naturally has a distinct magic number (0x01,0x32,0x1f).
