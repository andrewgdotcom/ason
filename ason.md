ASCII structured object notation
================================

ASCII structured object notation (ASON) is a non-binary serialised data format with the following unique properties:

* It is human-readable and -writable (with suitable editor support).
* It can be handled safely by non-ASON-aware text applications, including terminal emulators.
* It minimizes the need for escape sequences and quoting.
* Plaintext data can be trivially embedded inside an ASON document.
* Any ASON document can be trivially embedded inside another ASON document.

ASON does this by using the C0 information separator (IS) characters to impose a hierarchical structure on textual data, and the C0 transmission control (TC) characters to convey metatextual information.

ASON is a pure 7-bit ASCII encoding, and thus compatible with 8-bit extended-ASCII encodings such as ISO-2022 and UTF-8.
It can also contain arbitrary data via Base64 encoding.

ASON is to ASCII what REST is to HTTP.

Primitive encoding
------------------

ASON text may contain any byte that is not explicitly forbidden.
The seven forbidden characters have common special meanings when text is stored in memory or a file, or transmitted over a data link, and are therefore excluded on safety grounds.

* "ASON forbidden" - 0x00 [NUL], 0x04 [EOT], 0x11..0x14 [DC1-4], or 0x1a [SUB]

A further seven bytes are reserved to the ASON structure layer and have the same meaning regardless of context.

* "ASON structure" - 0x01..0x03 [SOH, STX, ETX] and 0x0c..0x0f [FS, GS, RS, US]

Two bytes indicate that leading or trailing data should be discarded by an ASON-aware application (see below).

* "ASON excisor" - 0x18, 0x19 [CAN, EM] (see below)

"ASON plaintext" is any other sequence of one or more bytes.
These are further classified:

* "Terminal-safe special value" - 0x05 [ENQ], 0x06 [ACK], 0x10 [DLE], or 0x15..0x17 [NAK, SYN, ETB] (see below)
* ASCII 0x07 [BEL] is treated as opaque
* "ASCII format effector" - 0x08..0x0d [BS, HT, LF, VT, FF, CR]
* ASCII 0x0e, 0x0f, 0x1b [SI, SO, ESC] are treated as per ISO-2022; any code-shift or escape sequences MUST be properly reverted or terminated.
* "ASCII graphical" - 0x20..0x7f, which take their usual ASCII meanings
* "extended-ASCII" - 0x80..0xff (encoding-dependent, treated as opaque)

### Special values

In ASON plaintext, the terminal-safe special values default to the following meanings:

* Boolean values are denoted by [ACK] (true) and [NAK] (false)
* Undefined values are denoted by [ENQ]
* Terminal-safe escape sequences are introduced by [DLE]
* Paragraph breaks are denoted by [ETB], as per IPTC-7901

ASON-aware applications MAY redefine these for their own purposes.

In addition, ASON considers [SYN] as a plaintext character, however its use is reserved to ASON itself.

### Excisors

[CAN, EM] are used to indicate the start and end of an ASON data stream embedded within another data stream.
Characters preceding a [CAN] and following an [EM] should be discarded by an ASON parser.

Note that excisor characters MAY appear inside ASON values; their interpretation will depend on whether ASON is expected or permitted in that context.

Structure encoding
------------------

A structure consists of a header text (htext) and an optional structured text (stext), delimited by [SOH, STX, ETX] = `^A ^B ^C` in the form:

    SOH htext [ STX stext ] ETX

The htext and stext consist of one or more fields, delimited by the information separators [FS, GS, RS, US] = `^\ ^] ^^ ^_` .

### htext

The htext is a sequence of one or more key-value records of the form:
    
    key US value [ RS key US value ... ]
    
where the field delimiters are [RS, US] = `^^ ^_`.

The "magic" key [SYN] = `^V` takes a plaintext value which indicates the [ASONDOC](asondoc.md) schema for an object structure.
If present, it SHOULD be the first record in the htext.
The value SHOULD be globally unique: values beginning with C0 control characters are reserved; reverse-domain format is RECOMMENDED for custom schemas.
A schema applies only to the current structure, and is not inherited by any structures embedded in its stext.

All other keys starting with [SYN] are reserved to ASON or ASONDOC.
Applications MAY define their own htext keys, but they and their values MUST NOT contain C0 characters.

### stext

The nature of the stext is flexible and application-dependent.

* If the stext is empty, the structure contains a single empty field.
* If the stext is missing, the structure contains no data.

An stext is represented as a hierarchical structure of fields

    field(1,1) [ US field(1,2) ... ] [ RS field(2,1) [ US field(2,2) ... ] ... ]

Fields MAY represent keys or values, depending on the schema in use.
An stext structure can have up to four dimensions by using all of [FS, GS, RS, US].

* Fields MUST either contain ASON text or be Base64 encoded (see below).
* The structure MAY be ragged-right.

Fields without values MAY contain the empty string, the `undefined` character [ENQ], or be missing entirely.
An application MAY distinguish between zero-length and undefined fields.

### Nesting

* A structure MAY be nested within another structure's stext.
* A structure MUST NOT be nested within an htext.
* All ASON texts used in the stext MUST be well-formed and properly nested.
* Any ISO-2022 locking-shift or escape sequences MUST be properly reverted or terminated at the end of each field, and before each embedded structure.

Any data that violates the above rules MUST be Base64 encoded (see below).

Note that child structures MUST NOT inherit any metadata (including ISO-2022 encodings) from their parent.
This ensures that ASON documents can be trivially embedded within each other without altering their semantics.

Field encoding
--------------

Data that is not valid ASON text MUST be Base64 encoded.
If a field's content contains a magic sequence, it indicates the encoding used for the subsequent bytes:

* [SYN ACK] = `^V ^F` base64
* [SYN NAK] = `^V ^U` none (default)

The Base64 encoding is as defined in RFC 2045, and embedded whitespace is permitted.
Embedded Base64 MUST be prefixed by [SYN ACK], which MAY appear anywhere in the plaintext.
Base64 is terminated by either an explicit [SYN NAK], or by the next ASON structure character, whichever comes first.

Concatenation of files
----------------------

An ASON file or data stream MAY contain one or more concatenated ASON structures at the top level; these MUST be interpreted as independent files.
For clarity, they SHOULD be separated by an [FS] character.
An [EM] character immediately following the outermost [ETX] character of an ASON document disables ASON interpretation for the rest of the file or data stream.
The meaning of any subsequent bytes is application-dependent.

An application SHOULD append [EM] to the end of the final (or only) ASON document in a file or data stream, and MUST do so if there is any subsequent non-ASON data.
An application MAY require a trailing [EM] to verify non-truncation of input data.
When concatenating multiple ASON files, any intervening non-ASON data, including excisors, MUST be removed.


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

The following control characters are understood by ASON at the metadata layer with these meanings, and MAY be used at the application layer (aside from [SYN]):

```
[ENQ] `^E` 0x05 (undefined)
[ACK] `^F` 0x06 (true)
[DLE] `^P` 0x10 (terminal-safe escape sequence)
[NAK] `^U` 0x15 (false)
[SYN] `^V` 0x16 (magic)
[ETB] `^W` 0x17 (paragraph)
```

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

ASONDOC
-------

[ASONDOC](asondoc.md) is a rich text format based on ASON.
The first header key [SYN] is mandatory, producing a reliable magic number [SOH SYN US] = `^A ^V ^_` (0x01,0x16,0x1f).

USON
----

ASON is an extended-ASCII format, and is agnostic about the particular extended-ASCII encoding used in the plaintext layer.
An application MAY transcode this to UTF-16 or UTF-32 for internal use, in which case all C0 control codes in the ASON structure MUST also be mapped to the corresponding UTF-16/32 codepoints, and the resulting format is called Unicode Structured Object Notation (USON).

If USON is written to persistent storage or passed between applications, then a byte-order mark (BOM) SHOULD be prepended to the beginning of the document, i.e. [BOM SOH SYN US ...].
This produces one of four ASONDOC magic numbers:

* 0xff 0xfe 0x01 0x00 0x16 0x00 0x1f 0x00 (UTF-16 little-endian)
* 0xfe 0xff 0x00 0x01 0x00 0x16 0x00 0x1f (UTF-16 big-endian)
* 0xff 0xfe 0x00 0x00 0x01 0x00 0x00 0x00 0x16 0x00 0x00 0x00 0x1f 0x00 0x00 0x00 (UTF-32 little-endian)
* 0x00 0x00 0xfe 0xff 0x00 0x00 0x00 0x01 0x00 0x00 0x00 0x16 0x00 0x00 0x00 0x1f (UTF-32 big-endian)

An ASON application SHOULD tolerate a UTF-8 BOM being prepended to UTF-8 encoded ASON.

ESON
----

It is also possible to define EBCDIC Structured Obect Notation by extension.
While ESON is not expected to be used natively by any application, it is defined so that a roundtrip conversion of ASCII->EBCDIC->ASCII remains well-behaved.

All of the C0 controls with meaning at the ASON layer have equivalents in EBCDIC, many of them with the same binary representation.
In the below, `**` indicates code points that differ.

Forbidden characters:

```
[NUL]    0x00
[DC1]    0x11
[DC2]    0x12
[DC3]    0x13
[DC4]    0x14
[EOT] ** 0x37 (ASCII 0x04)
[SUB] ** 0x3f (ASCII 0x1a)
```

Structure controls:

```
[SOH]    0x01 (start header)
[STX]    0x02 (start text)
[ETX]    0x03 (end text)
[FS]     0x1c (file separator)
[GS]     0x1d (group separator)
[RS]     0x1e (record separator)
[US]     0x1f (unit separator)
```

Excisors:

```
[CAN]    0x18 (start of data / ignore preceding data)
[EM]     0x19 (end of data / ignore trailing data)
```

Special values:

```
[ENQ] ** 0x2d (undefined)   (ASCII 0x05)
[ACK] ** 0x2e (true)        (ASCII 0x06)
[DLE]    0x10 (escape)
[NAK] ** 0x3d (false)       (ASCII 0x15)
[SYN] ** 0x32 (magic)       (ASCII 0x16)
[ETB] ** 0x26 (paragraph)   (ASCII 0x17)
```

Plaintext C0 controls:

```
[BEL] ** 0x2f (opaque)      (ASCII 0x07)
[BS]  ** 0x16 (format eff)  (ASCII 0x08)
[HT]  ** 0x05 (format eff)  (ASCII 0x09)
[LF]  ** 0x25 (format eff)  (ASCII 0x0a)
[VT]     0x0b (format eff)
[FF]     0x0c (format eff)
[CR]     0x0d (format eff)
[SI]     0x0e (ISO-2022)
[SO]     0x0f (ISO-2022)
[ESC] ** 0x27 (ISO-2022)    (ASCII 0x1b)
```

EBCDIC code points 0x15 [NEL] and 0x40..0xff are treated as plaintext.
The meaning of any other EBCDIC code point not listed above is undefined.

The representation of [SYN] differs between ASCII (0x16) and EBCDIC (0x32), so ESONDOC naturally has a distinct magic number (0x01,0x32,0x1f).


ASON-aware text display/edit
============================

An ASON-aware text display system SHOULD indicate ASON structure in a distinctive visual manner.
One simple method would be as follows:

* ASON structures are displayed outlined or distinctly coloured.
* [RS, US] delimited fields are arranged in 1-D or 2-D tables or arrays.
* [FS, GS] delimiters are indicated by hlines and/or distinct colours.
* Boolean values are shown using checkbox glyphs or similar, either inverse video or distinctly coloured.

An ASON-aware text input system SHOULD allow the user to enter ASON structure info in a standard manner:

* A typed Ctrl-A SHOULD indicate a new header and a typed Ctrl-B SHOULD end the header and start the text.
    * The editor MAY autocomplete the magic key and offer the user a menu of structure types.
    * The editor SHOULD autocomplete the trailing [ETX] so that the user does not have to type Ctrl-C.
* [FS, GS, RS, US] SHOULD be entered using Ctrl-4 to Ctrl-7

An ASON-aware IDE MAY also conform to the following:

* Characters or character sequences with special meaning in a particular application context MAY be mapped onto one of the ASON control characters, or an ASON structure.
* This mapping MAY be performed in real time if the application allows it.


References
==========

* ASCII is defined in ISO-646/ECMA-6: https://www.ecma-international.org/publications-and-standards/standards/ecma-6/
* C0 control codes are defined in ISO-6429/ECMA-48: https://www.ecma-international.org/publications-and-standards/standards/ecma-48/
* Transmission control characters TC1-TC10 are defined in ISO-1745/ECMA-16 : https://www.ecma-international.org/publications-and-standards/standards/ecma-16/
    * DLE sequences are defined in ECMA-37: https://www.ecma-international.org/publications-and-standards/standards/ecma-37/
* Base64 is defined in RFC-2045: https://datatracker.ietf.org/doc/html/rfc2045
* A similar application of C0 control characters can be found in IPTC-7901: https://www.iptc.org/std/IPTC7901/1.0/specification/7901V5.pdf
* A (joke?) encoding similar to ASON was previously proposed by Terence Eden: https://shkspr.mobi/blog/2017/03/kyli-because-it-is-superior-to-json/
