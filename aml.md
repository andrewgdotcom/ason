ASON Markup Language
====================

A standard method of marking up a document using printable characters, for rendering into [ASON](ason.md).
It is similar to (and inspired by) LaTeX and [SIL](https://sile-typesetter.org).
The main difference is that LaTeX and SIL are optimised for directly marking up human-readable and/or printable documents, while AML is optimised for marking up structured data.

Character Set
-------------

AML claims exclusive use of the `#` character.
It also claims use of the `\` character when prefixing ASCII alphanumerics or whitespace, but not when prefixing symbols.
To suppress interpretation of these characters, they SHOULD be escaped by a prefixed `\`.

AML also interprets square brackets and braces `[ ] { }` when directly prefixed by a macro name.
If they are not prefixed by a macro name and either appear in correctly-nested pairs, or if nesting correction is used (see below), then they are not interpreted.

Square brackets and braces are collectively referred to as "brackets" herein.

Macros
------

ASON structure is represented by a backslash-escaped macro name, followed by an optional square-bracket-delimited htext and one or more brace-delimited stext(s).

```
\name[htext]{stext}{...
```

The macro name is mapped onto the value of the magic htext record.

There MUST NOT be any whitespace before an opening bracket, otherwise it is not considered part of the macro.
Conversely, if a macro is followed directly by an opening bracket that is not intended as part of the macro, it MUST be separated by whitespace.
If inclusion of whitespace in the rendered ASON is not desired, it SHOULD be escaped by prefixing `\`.

```
\name{stext}\ {not stext}
```

### Environments

The special macros `\begin` and `\end` can be used to enclose larger stexts.
The name of the structure takes the place of the stext.

```
\begin[htext]{name}
stext
\end[htext]{name}
```

If an environment admits multiple stexts, they are separated (partitioned) by a `\part` macro.

```
\begin{name}
stext1
\part{name}
stext2
\end{name}
```

`\begin` and `\part` MUST be the last text on a line.
Newline characters immediately following a `\begin` or `\part` macro are considered part of the macro and removed.
The newline after the `\begin` macro is therefore not included in the stext, but the newlines before the `\end` and `\part` macros are.

`\part` and `\end` MUST be the first text on a line, apart from whitespace indentation.
Newline characters immediately preceding a `\part` or `\end` macro are NOT removed by default.
To suppress the newline before an `\end` or `\part`, append a `-` character to form `\end-` or `\part-`.

```
\begin{name}
stext1
\part-{name}
stext2
\end-{name}
```

The name given to `\end` or `\part` MUST match the one given to the corresponding `\begin`.
If any htext records other than `-` are given to `\end` or `\part` they MUST have also been given to `\begin`.
One or more `\end` or `\part` htext records MAY be omitted.

If a `\begin` macro is on a line by itself (before newline suppression, see below), and is indented by leading whitespace, all matching leading whitespace is removed from the following lines up to and including the corresponding `\end`.
Leading whitespace MUST match, including any whitespace before `\part` and `\end`.
To include leading whitespace where this is meaningful (such as block quotations) append a `=` character to form `\begin=`.

```
    \begin={name}
    stext with leading indentation and no trailing newline
    \end-{name}
```

### Elements

An element can be represented by a macro with no stext.

```
\element[params]
```

This is typically used for characters outside the ASCII range, or for additional markup.
Elements may be parameterised.

### Nesting Correction

All bracket pairs MUST be properly nested.
Square brackets MAY contain unpaired braces, and vice versa.

If an htext contains unpaired square brackets, they MUST be disambiguated by a "nester" consisting of one or more hash characters `#`.
The nester is placed before the htext, between the htext and stext, and after the stext.
The number of hash characters in the nester MUST be the same in each instance.
If there are multiple stexts, the nester MUST also appear between them, but MUST NOT be doubled.

```
\mymacro#[delimiter=]]#{]}#
```

If an stext contains unpaired braces, an environment MAY be used.
Otherwise, the nester SHOULD be employed.

```
\s##{}}##{\}}##
```

### Schema-free

The schema-free structure is represented by a macro with no name.
Its htext is optional, but its stext is not.

```
\{}
\[id=identifier]{}
```


Separators
----------

Information separators are represented directly by punctuation.

```
---\n   [FS]
;       [GS]
,       [RS]
:       [US]
```

The file separator `---` MUST appear by itself on a line, with no leading or trailing whitespace.

### Implicit Separators

A custom macro definition may specify an implicit separator for its stext(s).
An implicit separator is automatically inserted at each newline in an stext if:

* no explicit separator is present, and
* the newline is not suppressed by `\`

Typically, an implicit separator will be `,` (RS) or `;` (GS).

Plaintext
---------

ASON special plaintext values are represented by elements:

```
\true
\false
\null
```

An ASON paragraph break is indicated by a double-newline.

### Field Encoding

Base64 is enclosed by the special macro `\base64{...}`, or `\begin{base64}...\end{base64}`.
Despite its appearance it does not generate a structure, but is included directly into the surrounding text.

### AML Suppression

A custom macro definition can suppress AML macro processing within its stext(s).
If so, curly braces MUST still be properly nested.
All other uses of AML special characters are ignored.

### Excisions

Leading whitespace and trailing newlines are preserved using excisions.
This permits mostly-lossless roundtripping.

Leading indentation is preserved by inserting it in the following field as a leading excision, delimited by [CAN].
Trailing newlines (but not other trailing whitespace) are preserved as trailing excisions, delimited by [EM].

The newline character is always bare [LF], regardless of platform.
[CR] is treated as any other whitespace, and trimmed from the end of a line.

### Comments

A line comment is introduced by a hash character followed by a space, and continues until the next newline.
It is inserted into the previous field as a trailing excision, without the leading space but including the trailing newline.

```
some text, # this is a comment
more text
```

The above will be rendered as 

```
some text[EM]this is a comment[LF][RS]more text
```

To suppress interpretation of the hash character as a comment introducer, it should be escaped by `\`.

Multi-line comments are not directly supported, however applications may define their own comment-like structures.
