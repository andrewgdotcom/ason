ASCII structured object notation
================================

ASCII structured object notation (ASON) is a serialised data format similar to BSON or BJSON, but with the following unique properties:

* It is human-readable and -writable.
* It can be handled safely by non-ASON-aware text applications, including terminal emulators.
* It minimizes the need for escape sequences and quoting.
* Any commonly-used plaintext data can be trivially embedded inside an ASON document.
* Any ASON document can be trivially embedded inside another ASON document.

ASON does this by using the ISO-646 information separator characters to impose a hierarchical structure on textual data, and reusing the ISO-646 transmission control characters to convey metatextual information, similar to IPTC-7901.

* ISO-646: https://www.itscj.ipsj.or.jp/iso-ir/001.pdf
* IPTC-7901: https://www.iptc.org/std/IPTC7901/1.0/specification/7901V5.pdf

Primitive encoding
------------------

ASON may contain any byte that is not explicitly forbidden. A further eight bytes are reserved to the ASON structure layer and have the same meaning regardless of context.

* "ASON forbidden" - 0x00 [NUL], 0x11..0x14 [DC1-4], or 0x19..0x1a [CAN, EM, SUB] (but see note about EM below)
* "ASON structure" - 0x01..0x04 [SOH, STX, ETX, EOT] and 0x0c..0x0f [FS, GS, RS, US]

"ASON plaintext" is any other sequence of one or more bytes. These are further classified:

* "ASON special" - 0x05 [ENQ], 0x06 [ACK], 0x10 [DLE], or 0x15..0x17 [NAK, SYN, ETB]
* "ASCII whitespace" - 0x08 [BS], 0x09 [HT], or 0x20 [SP]
* "ASCII linebreak" - 0x0a..0x0d [LF, VT, FF, CR]
* "ASCII printable" - 0x21..0x7e, which take their usual ASCII meanings
* "ASCII extended" - 0x80..0xff (treated as opaque)
* ASCII 0x07, 0x0e, 0x0f, 0x1b, 0x7f [BEL, SI, SO, ESC, DEL] are treated as opaque, but any SI/SO or escape sequences SHOULD be properly terminated as per ISO-2022.

In ASON plaintext, the following meanings MAY be inferred if the application layer supports it:

* Boolean values MAY be encoded as [ACK] (true) and [NAK] (false).
* [ENQ] MAY be used to indicate "undefined".
* [ETB] MAY be used as a paragraph separator, as per IPTC-7901.
* Numeric values MAY be represented as a sequence of ASCII printable characters in one of the standard printf numeric formats.

Note that although [EM] is not itself a valid ASON character, it MAY be used to mark the end of an ASON section in a larger file.

Structure encoding
------------------

A structure consists of a header text (htext) and a structured text (stext), with an optional footer text (ftext), delimited by [SOH, STX, ETX, EOT] `^A^B^C^D` in the form:

    ^Ahtext^Bstext[^Cftext]^D

### htext and ftext

The htext is a sequence of one or more key/value records of the form `key^_value[^^key^_value[...]]` where the delimiters are [US, RS]:

* The key [SYN] `^V` MUST be present in the htext and MUST be the first key. It takes a special-character value that indicates the format of the structure (see below).
* The key [ETB] `^W` ("paragraph") MAY be present in the htext, in which case it MUST be the second key. It takes a value that contains the sequence of bytes that are used as the line terminator within the stext.
* The key = takes a plaintext value which is the name of the structure.
* The key - takes a plaintext value which is a caption for the structure. It may appear in either or both of the htext or ftext.

Applications may define their own plaintext keys, so long as they begin with an alphanumeric character. Keys with non-alphanumeric initials are reserved to ASON.

The ftext is similar to the htext, but MUST NOT contain the key [SYN].

### stext

The stext is a sequence of one or more ASON texts delimited by the information separators [FS, GS, RS, US] `^\^]^^^_` . The nature of the stext differs between structures.

### Nesting

A structure MAY be nested within another structure's stext. A structure MUST NOT be nested within an htext or ftext. All ASON texts used in the stext must be properly terminated and nested. Any [SO, SI] or other ISO-2022 sequences SHOULD be properly paired. Any values which violate ASON nesting or contain unsafe byte sequences MUST be sanitised, e.g. via ASCII armouring (application dependent).

Structured text formats
-----------------------

Structured text formats are denoted by a key-value pair where the key is [SYN] and the value is a transmission control character sequence.

* The stext of a quote [SOH SYN US DLE ENQ] `^A^V^_^P^E` ("ASON TC5") is a black box. Any enclosed ASON structure must properly nest but SHOULD NOT be interpreted.

* The stext of a list [SOH SYN US DLE ACK] `^A^V^_^P^F` ("ASON TC6") contains one or more fields of ASON text, separated by [US]. A list with a single entry can be used to create a file magic number where no other structure is required (see below).

* The stext of a dictionary [SOH SYN US DLE NAK] `^A^V^_^P^U` ("ASON TC8") is represented as `key1^_value1[^^key2^_value2...]`. It uses [US] to separate the keys from the values, and [RS] to delimit the records (this is the same structure as the htext and ftext). Empty keys are forbidden, and empty values should be indicated by [ENQ] (undefined). Otherwise, keys are plaintext and values are ASON text.

* The stext of a table [SOH SYN US DLE SYN] `^A^V^_^P^V` ("ASON TC9") is represented as `key1^_key2...^]value(1,key1)^_value(1,key2)...[^^value(2,key1)^_value(2,key2)...]`. The group separator [GS] is used to separate the first row, containing the column names (keys), from the rows (records) containing the values (fields). Empty keys are forbidden, and empty values should be indicated by [ENQ] (undefined). Otherwise, keys are plaintext and values are ASON text.

* The stext of an array [SOH SYN US DLE ETB] `^A^V^_^P^W` ("ASON TC10") is represented as `value(1,1)[^_value(1,2)...][^^value(2,1)[^_value(2,2)...]...]`. It can have up to four dimensions by using all of [FS, GS, RS, US]; up to sixteen by using the big-endian two-character sequences [FS FS, FS GS, FS RS,... US GS, US RS, US US]; up to 64 by using three-character sequences etc. Empty cells should be indicated by [ENQ] (undefined). Otherwise the cell contents are ASON text. Arrays MAY be ragged-right if the application allows it.

Magic number
------------

An ASON document MUST begin with the magic number [SOH SYN US] `^A^V^_` (0x01,0x16,0x1f). This is the start of an ASON structure, and is sufficient to uniquely identify an ASON document. [SYN] was chosen as the mandatory first key in order to produce a reliable magic number.

An ASON interpreter MAY search for this magic number at non-initial positions in a stream or file if the application allows it. For example, an ASON-aware script interpreter MAY ignore leading lines of the form `#!/path/to/executable\n`. IFF the magic number is found at a non-initial position it MUST be preceded by two SYN characters, i.e. [SYN SYN SOH SYN US] `^V^V^A^V^_`, and the interpretation of any preceding bytes is application-dependent. (cf IPTC-7901)

The presence of an [EM] character immediately following the [EOT] character of the outermost structure, i.e. [EOT EM] `^D^Y`, disables ASON interpretation for the rest of the file. The interpretation of any subsequent bytes is application-dependent. An application MAY require the [EOT EM] sequence to indicate non-truncation of data.


Compatibility and graceful degradation
======================================

ASCII has been a longstanding source of incompatibility:

* Linebreak sequences are either [CR, LF, CR LF] depending on context.
* Extended-ASCII documents may also use [NEL, LS, PS].
* Backspace may be used to indicate overstrike.

ASON has the potential to add further formatting chaos:

* Parsers may expect to process newline-terminated records, and may run out of buffer if newlines are not encountered frequently in tabular input.
* Legacy applications may not expect to process ASON, which may cause issues in parsing and display.
* Text viewers may not understand the ASON delimiters, leading to formatting issues.

We therefore explicitly instruct ASON about the nature of the linebreaks it should expect to find, and allow it to have embedded compatibility whitespace that formats the output when processed in legacy applications, but which ASON-aware applications can safely ignore.

* Any ASCII whitespace [BS, HT, SP] adjoining an information structure character [FS, GS, RS, US] MUST be ignored. Meaningful leading or trailing whitespace can be indicated by using a SYN character between the whitespace and the structure character, in which case only the SYN should be ignored.
* IFF the htext key [ETB] is provided, then its value MUST be a non-empty string containing ASCII linebreak bytes only. During decoding, any matching trailing string MUST be stripped from the last value of each record in all three texts of the structure, including in child structures unless overridden. Only one linebreak sequence should be stripped from each record.


ASON-aware text display/edit
============================

An ASON-aware text display system SHOULD conform to the following:

* [SOH, STX, ETX, EOT] delimited texts should be shown either inverse video or distinctly coloured.
* [RS, US] delimited structures should be shown as outlined 1-D or 2-D arrays.
    * Literal [RS, US] characters MAY be displayed as inverse-video or distinctly coloured `\n>` and `\t|` respectively.
* [FS, GS] delimited structures should be shown using hlines and/or distinct colours.
    * Literal [FS, GS] characters MAY be displayed as full lines of inverse-video or distinctly coloured `\n=====\n>` and `\n------\n>` respectively.
* Boolean values should be shown using checkbox glyphs or similar, either inverse video or distinctly coloured.

An ASON-aware text input system SHOULD in addition conform to the following:

* A typed Ctrl-A SHOULD indicate a new header and a typed Ctrl-B SHOULD end the header and start the text. The editor SHOULD then autocomplete the trailing [EOT] so that the user does not have to type it (given that Ctrl-D often causes connections to drop when typed).
* [FS, GS, RS, US] SHOULD be entered using Ctrl-4 to Ctrl-7

An ASON-aware IDE MAY also conform to the following:

* Characters or character sequences with special meaning in a particular application context MAY be mapped onto one of the ASON control characters, or an ASON structure.
* This mapping MAY be performed in real time if the application allows it.


Data representation
===================

At the data layer ASON is encoded as a sequence of bytes. Some of these are interpreted as ASCII control characters with special meaning to the structure and plaintext layers of ASON. All other bytes are either reserved to the plaintext layer, or forbidden.

C0 control chars reserved to the structure layer
------------------------------------------------

These bytes MUST NOT appear in the plaintext layer of ASON, as they are reserved to the structure of ASON.

```
[SOH] ^A 0x01 (start header)
[STX] ^B 0x02 (start text)
[ETX] ^C 0x03 (end text)
[EOT] ^D 0x04 (end footer)
...
[FS]  ^\ 0x1c (file separator)
[GS]  ^] 0x1d (group separator)
[RS]  ^^ 0x1e (record separator)
[US]  ^_ 0x1f (unit separator)
```

C0 control characters with special meanings in the plaintext layer
------------------------------------------------------------------

The following special meanings are understood by ASON at the metadata level, and MAY be used at the plaintext level if the application supports it:

```
[ENQ] ^E 0x05 (undefined)
[ACK] ^F 0x06 (true)
...
[DLE] ^P 0x10 (metadata)
...
[NAK] ^U 0x15 (false)
[SYN] ^V 0x16 (padding)
[ETB] ^W 0x17 (paragraph)
```

### DLE escape sequences

[DLE] is used to escape the normal meanings of [NAK, SYN, ETB] in order to encode structure metadata using terminal-safe unprintables:

```
[DLE ENQ] ^P^E 0x10,0x05 (TC5, quote)
[DLE ACK] ^P^F 0x10,0x06 (TC6, list)
...
[DLE NAK] ^P^U 0x10,0x15 (TC8, dictionary)
[DLE SYN] ^P^V 0x10,0x16 (TC9, table)
[DLE ETB] ^P^W 0x10,0x17 (TC10, array)
```

ISO-646 also permits the following unprintable sequences, which are either forbidden or reserved for future use:

```
[DLE SOH] ^P^A 0x10,0x01 (TC1, forbidden)
[DLE STX] ^P^B 0x10,0x02 (TC2, forbidden)
[DLE ETX] ^P^C 0x10,0x03 (TC3, forbidden)
[DLE EOT] ^P^D 0x10,0x04 (TC4, forbidden)
...
[DLE DLE] ^P^P 0x10,0x10 (TC7, reserved)
```

TC1-4 MUST NOT be used at any layer to avoid any ambiguity with ASON structure.

[DLE DLE] introduces an arbitrary-length string of ASON specials [ENQ, ACK, DLE, NAK, SYN, ETB], which terminates on the first character not from that set. No such sequences are currently defined.

C0 control characters with standard meanings in the plaintext layer
-------------------------------------------------------------------

The BEL, format effector and charset shift characters MAY appear in the plaintext layer of ASON, and have their usual meanings. If any escape sequences are used, they SHOULD be well-terminated and terminal-safe.

```
[BEL] ^G 0x07
[BS]  ^H 0x08 (whitespace)
[HT]  ^I 0x09 (whitespace)
[LF]  ^J 0x0a (linebreak)
[VT]  ^K 0x0b (linebreak)
[FF]  ^L 0x0c (linebreak)
[CR]  ^M 0x0d (linebreak)
[SI]  ^N 0x0e (charset)
[SO]  ^O 0x0f (charset)
...
[ESC] ^[ 0x1b
```

Plaintext may also contain any byte in the range 0x20..0xff.

C0 control characters forbidden
-------------------------------

These bytes MUST NOT appear anywhere in an ASON document. [EM] may be used to indicate the end of the document, but is not considered part of the document itself.

```
[NUL] ^@ 0x00
...
[DC1] ^Q 0x11
[DC2] ^R 0x12
[DC3] ^S 0x13
[DC4] ^T 0x14
...
[CAN] ^X 0x18
[EM]  ^Y 0x19
[SUB] ^Z 0x1a
```


ESON
====

It is also possible to define EBCDIC Structured Obect Notation by analogy with ASON. All of the C0 controls with meaning at the ASON layer have equivalents in EBCDIC, many of them at the same code points (indicated by **).

Structure controls:

```
[SOH] ** 0x01 (start header)
[STX] ** 0x02 (start text)
[ETX] ** 0x03 (end text)
[EOT]    0x37 (end footer)  (ASCII 0x04)
...
[FS]  ** 0x1c (file separator)
[GS]  ** 0x1d (group separator)
[RS]  ** 0x1e (record separator)
[US]  ** 0x1f (unit separator)
```

Special values:

```
[ENQ]    0x2d (undefined)   (ASCII 0x05)
[ACK]    0x2e (true)        (ASCII 0x06)
[DLE] ** 0x10 (escape)
[NAK]    0x3d (false)       (ASCII 0x15)
[SYN]    0x32 (padding)     (ASCII 0x16)
[ETB]    0x26 (paragraph)   (ASCII 0x17)
```

Format effectors:

```
[BEL]    0x2f               (ASCII 0x07)
[BS]     0x16 (whitespace)  (ASCII 0x08)
[HT]     0x05 (whitespace)  (ASCII 0x09)
[LF]     0x25 (linebreak)   (ASCII 0x0a)
[VT]  ** 0x0b (linebreak)
[FF]  ** 0x0c (linebreak)
[CR]  ** 0x0d (linebreak)
[SI]  ** 0x0e (charset)
[SO]  ** 0x0f (charset)
...
[ESC]    0x27               (ASCII 0x1b)
```

The representation of [SYN] differs between ASCII (0x16) and EBCDIC (0x32), so ESON naturally has a distinct magic number (0x01,0x32,0x1f).

While ESON is not expected to be used natively by any application, it is defined so that a roundtrip conversion of ASCII->EBCDIC->ASCII remains well-behaved.
