ASCII structured object notation
================================

ASCII structured object notation (ASON) is a non-binary serialised data format with the following unique properties:

* It is human-readable and -writable (with suitable editor support).
* It can be handled safely by non-ASON-aware text applications, including terminal emulators.
* It minimizes the need for escape sequences and quoting.
* Plaintext data can be trivially embedded inside an ASON document.
* Any ASON document can be trivially embedded inside another ASON document.

ASON does this by using the C0 information separator characters to impose a hierarchical structure on textual data, and reusing the C0 transmission control characters to convey metatextual information, similar to IPTC-7901.

ASON is a pure 7-bit ASCII encoding, and is therefore automatically compatible with 8-bit extended-ASCII encodings such as ISO-2022 and UTF-8.

References
----------

* ASCII is defined in ISO-646/ECMA-6: https://www.ecma-international.org/publications-and-standards/standards/ecma-6/
* C0 control codes are defined in ISO-6429/ECMA-48: https://www.ecma-international.org/publications-and-standards/standards/ecma-48/
* Transmission control characters TC1-TC10 are defined in ISO-1745/ECMA-16 : https://www.ecma-international.org/publications-and-standards/standards/ecma-16/
    * DLE sequences (TC7) are defined in ECMA-37: https://www.ecma-international.org/publications-and-standards/standards/ecma-37/
    * A similar application of transmission control characters can be found in IPTC-7901: https://www.iptc.org/std/IPTC7901/1.0/specification/7901V5.pdf

Primitive encoding
------------------

ASON may contain any byte that is not explicitly forbidden.
The six forbidden characters have common special meanings when text is stored in memory or a file, or transmitted over a data link, and are therefore excluded.

* "ASON forbidden" - 0x00 [NUL], 0x11..0x14 [DC1-4], or 0x1a [SUB]

A further eight bytes are reserved to the ASON structure layer and have the same meaning regardless of context.

* "ASON structure" - 0x01..0x04 [SOH, STX, ETX, EOT] and 0x0c..0x0f [FS, GS, RS, US]

Two bytes indicate that leading or trailing data should be discarded by an ASON-aware application (see below).

* "ASON excisor" - 0x18, 0x19 [CAN, EM]

"ASON plaintext" is any other sequence of one or more bytes.
These are further classified:

* "ASON special value" - 0x05 [ENQ], 0x06 [ACK], 0x10 [DLE], or 0x15..0x17 [NAK, SYN, ETB] (see below)
* ASCII 0x07 [BEL] is treated as opaque
* "ASCII format effector" - 0x08..0x0d [BS, HT, LF, VT, FF, CR]
* ASCII 0x0e, 0x0f, 0x1b [SI, SO, ESC] are treated as per ISO-2022, but any SI/SO or escape sequences MUST be properly terminated, and the C0 codes MUST NOT be paged out at any time.
* "ASCII graphical" - 0x20..0x7f, which take their usual ASCII meanings
* "extended-ASCII" - 0x80..0xff (encoding-dependent, treated as opaque)

### Special values

In ASON plaintext, the following standard meanings are defined:

* Boolean values are encoded as [ACK] (true) and [NAK] (false)
* Undefined values are encoded as [ENQ]
* Paragraph breaks are encoded as [ETB], as per IPTC-7901

### Excisors

* [CAN] is used to indicate the start of an ASON value; any data before it in the current field MUST be discarded.
* [EM] is used to indicate the end of an ASON value; any data after it in the current field MUST be discarded.

When used outside a field, [EM] indicates the end of an ASON document.

Structure encoding
------------------

A structure consists of a header text (htext) and a structured text (stext), with an optional footer text (ftext), delimited by [SOH, STX, ETX, EOT] = `^A ^B ^C ^D` in the form:

    SOH htext STX stext [ ETX ftext ] EOT

### htext and ftext

The htext is a sequence of one or more key/value records of the form:
    
    key US value [ RS key US value ... ]
    
where the delimiters are [RS, US] = `^^ ^_`.

* The magic key [SYN] = `^V` MUST be present in the htext and MUST be the first key.
    It takes a value that indicates the format of the structure (see below).
* The key [SYN =] = `^V =` takes a plaintext value which is the name of the structure.
* The key [SYN -] = `^V -` takes a plaintext value which is a caption for the structure.
    It may appear in either or both of the htext or ftext.

Keys starting with [SYN] are reserved to ASON.
Applications may define their own htext keys, so long as they contain only ASCII graphical or extended-ASCII characters.

The ftext is similar to the htext, but MUST NOT contain the magic key [SYN].

### stext

The stext is a sequence of one or more ASON texts delimited by the information separators [FS, GS, RS, US] = `^\ ^] ^^ ^_` .
The nature of the stext differs between structures.

### Nesting

A structure MAY be nested within another structure's stext.
A structure MUST NOT be nested within an htext or ftext.
All ASON texts used in the stext must be well-formed and properly nested.
Any [SO, SI] or ISO-2022 escape sequences MUST be properly paired and terminated, and the C0 character set MUST NOT be paged out at any time.
Any values which violate the above nesting rules or contain unsafe byte sequences MUST be sanitised, e.g. via BASE64 encoding (application dependent).

Note that child structures do not inherit any metadata from their parent.
This ensures that ASON documents can be trivially embedded within each other without altering their semantics.

Structured text formats
-----------------------

Structured text formats are denoted by a key-value pair where the key is [SYN] and the value is a metadata string.
The metadata string consists of a [DLE] character followed by a character from the set [ENQ, ACK, DLE, NAK, SYN, ETB].

### Quote

The stext of a quote [SOH SYN US DLE ENQ] = `^A ^V ^_ ^P ^E` is opaque.
Any enclosed ASON structure MUST properly nest but SHOULD NOT be interpreted.

### List

The stext of a list [SOH SYN US DLE ACK] = `^A ^V ^_ ^P ^F` contains one or more fields of ASON text, separated by [US].
A list with a single entry can be used to create a file magic number where no other structure is required (see below).
If the list has no entries, then its stext SHOULD consist only of [ENQ].

### Object

The stext of an object [SOH SYN US DLE DLE] = `^A ^V ^_ ^P ^P` is application-defined.
It MAY use any combination of [FS, GS, RS, US] to delimit its stext, but MUST conform to ASON nesting rules.

### Dictionary

The stext of a dictionary [SOH SYN US DLE NAK] = `^A ^V ^_ ^P ^U` is represented as

    key1 US value1 [ RS key2 US value2 ... ]

It uses [US] to separate the keys from the values, and [RS] to delimit the records (this is the same structure as the htext and ftext).
Empty keys are forbidden, and empty values MUST be indicated by [ENQ] (undefined).
Otherwise, keys are plaintext and values are ASON text.
If the dictionary has no keys, then its stext SHOULD consist only of [ENQ].

### Table

The stext of a table [SOH SYN US DLE SYN] = `^A ^V ^_ ^P ^V` is represented as
    
    key1 US key2 ... GS value(1,key1) US value(1,key2) ... [ RS value(2,key1) US value(2,key2) ... ]

The group separator [GS] is used to separate the first row, containing the column names (keys), from the rows (records) containing the values (fields).
Empty keys are forbidden, and empty values MUST be indicated by [ENQ] (undefined).
Otherwise, keys are plaintext and values are ASON text.
If the table has keys but no records, then it SHOULD contain a single value [ENQ].
If the table has no keys, then its stext SHOULD consist only of [ENQ].

### Array

The stext of an array [SOH SYN US DLE ETB] = `^A ^V ^_ ^P ^W` is represented as

    value(1,1) [ US value(1,2) ... ] [ RS value(2,1) [ US value(2,2) ... ] ... ]

It can have up to four dimensions by using all of [FS, GS, RS, US]; up to sixteen by using the big-endian two-character sequences [FS FS, FS GS, FS RS,... US GS, US RS, US US]; up to 64 by using three-character sequences etc.
Separator sequences of differing lengths MUST NOT be used in the same array.
Empty cells MUST be indicated by [ENQ] (undefined), otherwise the cell contents are ASON text.
Arrays MAY be ragged-right if the application allows it.
If the array has no entries, then its stext SHOULD consist only of [ENQ].


Compatibility
=============

Identification
--------------

An ASON document MUST begin with the magic number [SOH SYN US] = `^A ^V ^_` (0x01,0x16,0x1f).
This is the start of an ASON structure, and is sufficient to uniquely identify an ASON document.
[SYN] was chosen as the mandatory first key in order to produce a reliable magic number.

An ASON interpreter MAY search for this magic number at non-initial positions in a stream or file if the application allows it.
For example, an ASON-aware script interpreter MAY ignore leading lines of the form `#!/path/to/executable\n`.
IFF the magic number is found at a non-initial position it MUST either:

* be preceded a CAN character, i.e. [CAN SOH SYN US] = `^X ^A ^V ^_`, and the interpretation of any preceding bytes is application-dependent.
* be preceded only by a UTF-8 encoded byte-order mark (BOM) (see "USON" below).

The presence of an [EM] character immediately following the [EOT] character of the outermost structure, i.e. [EOT EM] = `^D ^Y`, disables ASON interpretation for the rest of the file.
The meaning of any subsequent bytes is application-dependent.
An application SHOULD append [EM] to the end of an emitted ASON document, and MUST do so if there are any subsequent bytes in the file.
An application MAY require a trailing [EM] to verify non-truncation of input data.

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

    structure [ plaintext CAN ] value [ EM plaintext ] structure

For example, the plaintext preceding [CAN] might contain whitespace indentation, and the plaintext following [EM] might contain a line break.
This allows a non-ASON editor or terminal to display structured data in human-readable form, while the ASON structure encodes the same information in machine-readable form.

ASON-aware text display/edit
----------------------------

An ASON-aware text display system SHOULD indicate ASON structure in a distinctive visual manner.
One suggested (crude) method is as follows:

* [SOH, STX, ETX, EOT] delimited texts are shown either inverse video or distinctly coloured.
* [RS, US] delimited structures are shown as outlined 1-D or 2-D arrays.
    * Literal [RS, US] characters may be displayed as inverse-video or distinctly coloured `\n>` and `\t|` respectively.
* [FS, GS] delimited structures are shown using hlines and/or distinct colours.
    * Literal [FS, GS] characters may be displayed as full lines of inverse-video or distinctly coloured `\n=====\n>` and `\n------\n>` respectively.
* Boolean values are shown using checkbox glyphs or similar, either inverse video or distinctly coloured.

An ASON-aware text input system SHOULD allow the user to enter ASON structure info in a standard manner:

* A typed Ctrl-A SHOULD indicate a new header and a typed Ctrl-B SHOULD end the header and start the text.
    * The editor SHOULD autocomplete the magic key and offer the user a menu of structure types.
    * The editor SHOULD autocomplete the trailing [EOT] so that the user does not have to type it (given that Ctrl-D often causes connections to drop when typed).
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
[EOT] `^D` 0x04 (end footer)
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

As per ECMA-37, [DLE] is used to escape the normal meanings of [ENQ, ACK, DLE, NAK, SYN, ETB].
ASON uses these sequences to encode structure metadata using terminal-safe C0 codes:

```
[DLE ENQ] `^P ^E` 0x10,0x05 (quote)
[DLE ACK] `^P ^F` 0x10,0x06 (list)
[DLE DLE] `^P ^P` 0x10,0x10 (object)
[DLE NAK] `^P ^U` 0x10,0x15 (dictionary)
[DLE SYN] `^P ^V` 0x10,0x16 (table)
[DLE ETB] `^P ^W` 0x10,0x17 (array)
```

ISO-646 also permits the following sequences, but these are forbidden in ASON (even at the application layer) to avoid any ambiguity with ASON structure:

```
[DLE SOH] `^P ^A` 0x10,0x01
[DLE STX] `^P ^B` 0x10,0x02
[DLE ETX] `^P ^C` 0x10,0x03
[DLE EOT] `^P ^D` 0x10,0x04
```

C0 control characters with standard meanings in the plaintext layer
-------------------------------------------------------------------

The BEL, format effector and charset shift characters MAY appear in the plaintext layer of ASON, and have their usual meanings.
If any escape sequences are used, they SHOULD be well-terminated and terminal-safe.

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

All of the C0 controls with meaning at the ASON layer have equivalents in EBCDIC, many of them with the same binary values (indicated by **).

Forbidden characters:

```
[NUL] ** 0x00
[DC1] ** 0x11
[DC2] ** 0x12
[DC3] ** 0x13
[DC4] ** 0x14
[SUB]    0x3f (ASCII 0x1a)
```

Structure controls:

```
[SOH] ** 0x01 (start header)
[STX] ** 0x02 (start text)
[ETX] ** 0x03 (end text)
[EOT]    0x37 (end footer)  (ASCII 0x04)
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
