# General Transformation Format 8 (GTF-8)

UTF-8 is a variable-width, self-synchronising encoding of unsigned integer values representing code points of the Universal Character Set (UCS).
It has several limitations specific to UCS, such as an upper bound of 0x10FFFF and various forbidden values, making it unsuitable for general use.

We are therefore motivated to define a generalised encoding framework with similar properties, but without UCS-specific limitations.

# GTF-8 Framework

Any unsigned integer value may be encoded in GTF-8.
Arbitrarily large values may be encoded, bounded only by available resources.
Other data types may be represented in GTF-8 if they can be mapped onto the unsigned integers.
Such mappings are outside the scope of this document.

## Sequences

Each encoded value is represented by a sequence of one or more bytes.
Each byte value comes from one of three bitmask ranges, the extents of which are tunable.
One of the ranges can be at most 7 bits wide, while the other two are at most 6 bits wide.
Each range carries N bits of *payload* in its least-significant bits, with the 8-N most-significant bits fixed.

* single-byte sequence (N=S bits wide)
* initial byte of multi-byte sequence (N=I bits wide)
* continuation byte of multi-byte sequence (N=C bits wide)

Encoded values that require at most S bits are represented directly by the S payload bits of a single byte.
Larger values are represented by a multi-byte sequence.

Each multi-byte sequence consists of one initial byte, followed by one or more continuation bytes.
The length of the sequence is determined by the value of the initial byte, unless the initial byte has the maximum permitted value in its range.
If so, the length is determined by the value of the second byte, unless it has the maximum permitted value, and so on.

## Payload Structure

Each initial byte carries I bits of payload in its least-significant bits.
Each continuation byte carries C bits of payload in its least-significant bits.
If L is the number of continuation bytes in the sequence, the total payload consists of L*C+I bits.

The payload bits are concatenated, then split between a length prefix and the encoded data:

* L bits of ones
* 1 bit of zero
* L*(C-1)+I-1 bits of encoded data

The sequence length is uniquely determined by the position of the first zero in the payload bits.
Each continuation byte adds C bits of payload, but is offset by an additional bit of length prefix, so adds a net C-1 bits of encoded data.

## Comparison with BASE-64

Unlike BASE-64, a GTF-8 encoder must know the total number of bits to be encoded in advance.
This makes it unsuitable for direct representation of streaming data.
Instead the data stream may be chunked, with prefixed GTF-8 values used to represent the length of each following chunk.


# Variants

The GTF-8 framework permits multiple specific encoding schemes, called *variants*.
Each variant is defined by specifying the set of byte ranges in use.
Two variants are currently defined, GTF-8U and GTF-8N.

(The term "GTF-8X" is used below to indicate a particular but unspecified variant.)


## GTF-8U

General Transformation Format 8U is a superset of UTF-8, and its byte ranges correspond to those of UTF-8:

* 0x00..0x7F - single-byte sequence
* 0x80..0xBF - continuation byte of multi-byte sequence
* 0xC0..0xFF - initial byte of multi-byte sequence

This ensures there are 128 single-byte encodings for backwards compatibility with US-ASCII.

Encoded values less than 128 (2^7) are represented by a single byte of the same value.
All other values are represented by multi-byte sequences.

| Initial Byte  | Bytes in Sequence | Bits Encoded | Maximum Value
|---------------|-------------------|--------------|--------------
| 0x00..0x7F    | 1                 | 7            | 0x7F
| 0xC0..0xDF    | 2                 | 11           | 0x07FF
| 0xE0..0xEF    | 3                 | 16           | 0xFFFF
| 0xF0..0xF7    | 4                 | 21           | 0x1FFFFF
| 0xF8..0xFB    | 5                 | 26           | 0x03FFFFFF
| 0xFC..0xFD    | 6                 | 31           | 0x7FFFFFFF
| 0xFE          | 7                 | 36           | 0x0FFFFFFFFF
| 0xFF80..0xFF9F| 8                 | 41           | 0x01FFFFFFFFFF
| 0xFFA0..0xFFAF| 9                 | 46           | 0x3FFFFFFFFFFF
| 0xFFB0..0xFFB7| 10                | 51           | 0x07FFFFFFFFFFFF
| 0xFFB8..0xFFBB| 11                | 56           | 0xFFFFFFFFFFFFFF
| 0xFFBC..0xFFBD| 12                | 61           | 0x1FFFFFFFFFFFFFFF
| 0xFFBE        | 13                | 66           | 0x03FFFFFFFFFFFFFFFF
| 0xFFBF80..0xFFBF9F | 14           | 71           | 0x7FFFFFFFFFFFFFFFFF
| 0xFFBFA0..0xFFBFAF | 15           | 76           | 0x0FFFFFFFFFFFFFFFFFFF

Sequences of four bytes or less are encoded in the same format as UTF-8.
Sequences of five or six bytes correspond to the now-obsolete specification of UTF-8 in RFC2279.

Each additional continuation byte adds 6 payload bits, however one leading bit of the payload is consumed for each additional continuation byte.
Each additional continuation byte therefore adds 5 bits of encoding capacity.

0yyyzzzz
110xxxyy 10yyzzzz
1110wwww 10xxxxyy 10yyzzzz
11110uvv 10vvwwww 10xxxxyy 10yyzzzz
111110tt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
1111110s 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
11111110 10rrrrss 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
11111111 100pqqqq 10rrrrss 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
11111111 1010oopp 10ppqqqq 10rrrrss 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
11111111 10110nnn 10oooopp 10ppqqqq 10rrrrss 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
11111111 101110mm 10mmnnnn 10oooopp 10ppqqqq 10rrrrss 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
11111111 1011110k 10llllmm 10mmnnnn 10oooopp 10ppqqqq 10rrrrss 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
11111111 10111110 10jjkkkk 10llllmm 10mmnnnn 10oooopp 10ppqqqq 10rrrrss 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
11111111 10111111 100iiijj 10jjkkkk 10llllmm 10mmnnnn 10oooopp 10ppqqqq 10rrrrss 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz
11111111 10111111 1010hhhh 10iiiijj 10jjkkkk 10llllmm 10mmnnnn 10oooopp 10ppqqqq 10rrrrss 10sstttt 10uuuuvv 10vvwwww 10xxxxyy 10yyzzzz

The maximum information density of a GTF-8U multi-byte sequence is (5n+1)/8n, where n is the number of bytes in the sequence.
This is achieved when the number of bits to be encoded matches the maximum number of bits encoded by n bytes, and no padding with leading zero bits is required.
The minimum information density is found one bit beyond the maximum density, and is (5n-4)/8n.
In the large limit the maximum information density approaches 5/8 from above, while the minimum information density approaches it from below.

As a superset, all UTF-8 byte sequences are valid GTF-8U, but the converse is not true.


## GTF-8N

If US-ASCII compatibility is not required, we can optimise for a higher and more consistent information density, at the expense of providing fewer one-byte encodings.
We can also reorder the byte ranges to simplify the encoding and decoding algorithms.

In the GTF-8N encoding scheme, single-byte encodings and initial bytes have the most significant bit set, and continuation bytes have it unset.
This effectively swaps the single-byte and continuation byte ranges when compared with GTF-8U:

* 0x00..0x7F - continuation byte of multi-byte sequence
* 0x80..0xBF - single-byte sequence
* 0xC0..0xFF - initial byte of multi-byte sequence

This interchange means that it is not possible for a byte sequence to be both valid GTF-8U and GTF-8N.

As in GTF-8U, the number of bytes in a multi-byte sequence is given by the number of leading 1 bits in the initial byte, but this now also applies to single-byte encodings, which have a single leading 1 bit.
The single-byte and initial-byte ranges are thereby unified, and single-byte sequences can be viewed as a limiting case of multi-byte sequences.
Continuation bytes have no leading 1 bits, or equivalently they have a leading zero bit, leaving seven bits free for encoded values.

| Initial Bytes  | Bytes in Sequence | Bits Encoded | Maximum Value
|----------------|-------------------|--------------|---------------
| 0x80..0xBF     | 1                 | 6            | 0x3F
| 0xC0..0xDF     | 2                 | 12           | 0x0FFF
| 0xE0..0xEF     | 3                 | 18           | 0x03FFFF
| 0xF0..0xF7     | 4                 | 24           | 0xFFFFFF
| 0xF8..0xFB     | 5                 | 30           | 0x3FFFFFFF
| 0xFC..0xFD     | 6                 | 36           | 0x0FFFFFFFFF
| 0xFE           | 7                 | 42           | 0x03FFFFFFFFFF
| 0xFF00..0xFF3F | 8                 | 48           | 0xFFFFFFFFFFFF
| 0xFF40..0xFF5F | 9                 | 54           | 0x3FFFFFFFFFFFFF
| 0xFF60..0xFF6F | 10                | 60           | 0x0FFFFFFFFFFFFFFF
| 0xFF70..0xFF77 | 11                | 66           | 0x03FFFFFFFFFFFFFFFF
| 0xFF78..0xFF7B | 12                | 72           | 0xFFFFFFFFFFFFFFFFFF
| 0xFF7C..0xFF7D | 13                | 78           | 0x3FFFFFFFFFFFFFFFFFFF
| 0xFF7E         | 14                | 84           | 0x0FFFFFFFFFFFFFFFFFFFFF
| 0xFF7F00..0xFF7F3F | 15            | 90           | 0x03FFFFFFFFFFFFFFFFFFFFFF

Each additional continuation byte adds 7 payload bits, with one leading bit of the payload consumed for each additional continuation byte.
Each additional continuation byte therefore adds 6 bits of encoding capacity.

10yyzzzz
110xxxxy 0yyyzzzz
1110vvww 0wwxxxxy 0yyyzzzz
11110uuu 0uvvvvww 0wwxxxxy 0yyyzzzz
111110ss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
1111110r 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
11111110 0ppqqqqr 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
11111111 00oooopp 0ppqqqqr 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
11111111 010mmnnn 0noooopp 0ppqqqqr 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
11111111 0110llll 0mmmmnnn 0noooopp 0ppqqqqr 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
11111111 01110jjk 0kkkllll 0mmmmnnn 0noooopp 0ppqqqqr 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
11111111 011110ii 0iijjjjk 0kkkllll 0mmmmnnn 0noooopp 0ppqqqqr 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
11111111 0111110g 0ghhhhii 0iijjjjk 0kkkllll 0mmmmnnn 0noooopp 0ppqqqqr 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
11111111 01111110 0ffffggg 0ghhhhii 0iijjjjk 0kkkllll 0mmmmnnn 0noooopp 0ppqqqqr 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz
11111111 01111111 00ddeeee 0ffffggg 0ghhhhii 0iijjjjk 0kkkllll 0mmmmnnn 0noooopp 0ppqqqqr 0rrrssss 0ttttuuu 0uvvvvww 0wwxxxxy 0yyyzzzz

This results in a maximum information density of 6/8, the same as BASE-64, while the minimum information density is (6n-5)/8n.


## Comparison of GTF-8U and 8N

### Information Density

GTF-8N has equal or higher information density than GTF-8U for all encodable values greater than 127, and equal density for all values less than 64, which can be encoded in a single byte in both GTF-8U and 8N.
Seven-bit values (64..127) are the only ones for which GTF-8U provides a shorter encoding.
This is because seven bits is the point where GTF-8U reaches its maximum information density of 87.5% and GTF-8N its minimum density of 43.75%.

It should be noted however that for most values the difference between the two encodings is not significant.

All but one bit length (other than 12) up to 16 bits, and the majority of bit lengths up to and including 31 bits, have identical encoding lengths in both GTF-8U and 8N.
31-bit values are the largest with the same sequence lengths (6 bytes) in both GTF-8U and 8N.
All values of 32 bits or more have a GTF-8U representation at least one byte longer than GTF-8N.
42-bit values are the smallest where the sequence lengths differ by more than one byte.

This means that when dealing with values of less than 32 bits, there is only a sporadic efficiency advantage when using 8N rather than 8U.
Only for 32-bit values and above does GTF-8N have a consistent advantage, with a gain of between 16 and 20%.

| Bits  | 8U B  | 8U %  | 8N B  | 8N %  | ratio
|-------|-------|-------|-------|-------|-------
| ..    | | | | | |
| 6     | 1     | 75    | 1     | 75    | 1
| 7     | 1     | 87.5  | 2     | 43.75 | 0.5
| 8     | 2     | 50    | 2     | 50    | 1
| 9     | 2     | 56.25 | 2     | 56.25 | 1
| 10    | 2     | 62.5  | 2     | 62.5  | 1
| 11    | 2     | 68.75 | 2     | 68.75 | 1
| 12    | 3     | 50    | 2     | 75    | 1.5
| 13    | 3     | 54.16 | 3     | 54.16 | 1
| 14    | 3     | 58.333| 3     | 58.333| 1
| 15    | 3     | 62.5  | 3     | 62.5  | 1
| 16    | 3     | 66.667| 3     | 66.667| 1
| 17    | 4     | 53.125| 3     | 70.833| 1.333
| 18    | 4     | 56.25 | 3     | 75    | 1.333
| 19    | 4     | 59.375| 4     | 59.375| 1
| 20    | 4     | 62.5  | 4     | 62.5  | 1
| 21    | 4     | 65.625| 4     | 65.625| 1
| 22    | 5     | 55    | 4     | 68.75 | 1.25
| 23    | 5     | 57.5  | 4     | 71.875| 1.25
| 24    | 5     | 60    | 4     | 75    | 1.25
| 25    | 5     | 62.5  | 5     | 62.5  | 1
| 26    | 5     | 65    | 5     | 65    | 1
| 27    | 6     | 56.25 | 5     | 67.5  | 1.2
| 28    | 6     | 58.333| 5     | 70    | 1.2
| 29    | 6     | 60.417| 5     | 72.5  | 1.2
| 30    | 6     | 62.5  | 5     | 75    | 1.2
| 31    | 6     | 64.583| 6     | 64.583| 1
| 32    | 7     | 57.143| 6     | 66.667| 1.167
| ..    | | | | |
| 64    | 13    | 61.538| 11    | 72.727| 1.182
| ..    | | | | |
| 96    | 19    | 63.157| 16    | 75    | 1.188
| ..    | | | | |
| 128   | 26    | 61.538| 22    | 72.727| 1.182
| ..    | | | | |
| limit |       | 62.5  |       | 75    | 1.2

It is also apparent that beyond 32 bits, chunks of raw data prefixed with GTF-8 byte lengths become equally or more efficient than GTF-8 itself, with an efficiency approaching 100% in the limit.
Multiplexing (see below) can be used to distinguish between directly-encoded values and chunk length prefixes.

### (In)compatibility with UTF-8

Since GTF-8U is a superset of UTF-8, naive software may mistake it for UTF-8 and try to display it as text.
All sequences of GTF-8N bytes are incompatible with UTF-8 and cannot be confused with it, which may be considered a feature.


# Multiplexing

An encoded value must be padded with leading zero bits to the next nearest encodable bit length, however it is also possible to pad to any greater bit length.
An overlong encoding is one where the encoded value has been padded with more zeros than strictly required, to produce a sequence with an excess number of continuation bytes.

By interpreting the number of excess continuation bytes in an overlong encoding as an embedded tag, a single GTF-8 data stream can be multiplexed into distinct channels.
Tag 0 (with no overlong encodings) would be interpreted as the default or "data" channel.
If used, tag 1 (with 1 excess byte per sequence) would represent the first "metadata" channel, and so on for tag 2, 3, etc.

Each channel is successively less efficient in the number of bits encoded per byte, although the efficiency of all channels approaches the same value in the large limit.

One use case for multiplexing would be a data format where channel 0 encodes small values directly, while channel 1 indicates the length of a raw data chunk that follows.
For values greater than 31 bits in GTF-8U, or 54 bits in GTF-8N, the efficiency gained from using raw chunks more than offsets the minimum two-byte cost incurred by an overlong length prefix.
It would therefore be reasonable for a data format expecting to handle (say) 64-bit numbers to permit the use of channel-1-length-prefixed chunks instead of native GTF-8.


# Profiles

We specify three standard profiles of each GTF-8X, depending on their treatment of overlong encodings:

* GTF-8Xs (strict) forbids overlong encodings
* GTF-8Xp (permissive) allows overlong encodings of any value
* GTF-8Xm (multiplexed) uses overlong encodings to denote multiplexed channel tags

Equivalently, GTF-8Xs can be defined as "channel tags >0 are forbidden", and GTF-8Xp as "channel tags are ignored".

## Custom Profiles

Applications may define custom profiles that specify the use of overlong encodings on a granular basis, and/or apply other restrictions such as maximum values.
UTF-8 can be interpreted as a custom profile of GTF-8Us which also applies the maximum and forbidden values of UCS.
