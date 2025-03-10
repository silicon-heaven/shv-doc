# CPON - ChainPack Object Notation

CPON is inspired by [JSON](https://www.json.org/json-en.html). It is used for
data dumping as well as user input.

CPON text should be encoded in UTF-8.

CPON supports C style comments and thus `/* foo */` is just simply ignored and
considered as a white space character.

[RPC Value](./rpcvalue.md) types are encoded in the following way:

### Null
Represented with plain `null` (compatible with JSON).

### Bool
Represented with either `true` or `false` (compatible with JSON).

### Int
Represented as number just like in JSON (e.g. `123`, `-42`) with additional
support for common hexadecimal (e.g. `0x20`) and binary representation (e.g.
`0b1001`).

Tools generating CPON should prefer plain number representation (thus compatible
with JSON).

### UInt
Represented as **Int** with suffix `u` (e.g. `123u`, `0x20u`, `0b1001u`).

### Double
Represented in scientific notation with `P` (or `p`) between significant and
exponent represented as **Int** (e.g.  `1.25p-2`, `-0.0625p3`, `0b1001p+2`).

Tools generating CPON should prefer format `[-]0xh.hhhp[+|-]dd`, thus
significant is normalized hexadecimal number and exponent is decimal.

### Decimal
Represented as decimal numbers with decimal point (`.`) or in scientific E
notation (both `E` and `e` can be used) (e.g. `123.45`, `1.2345e2`, `12345E-2`).

Tools generating CPON should prefer scientific notation with `e`  and optionally
representation with decimal point if number of inserted zeroes would be small
enough.

### Blob
Represented as **String** with `b` prefix where values from `0x00` to `0x1f` and
from `0x7f` to `0xff` are represented as escape sequences `\hh` (where `hh` is
character's hexadecimal numeric value) with these exceptions:

| Character | Escape sequence |
|-----------|-----------------|
| `\`       | `\\`            |
| `"`       | `\"`            |
| TAB       | `\t`            |
| CR        | `\r`            |
| LF        | `\n`            |

Example: `b"ab\31"`

Tools generating CPON should prefer special escape sequences over hexadecimal
numeric ones.

### HexBlob
This is CPON extension on **Blob** type. There is no such alternative in
ChainPack.

Blob is represented as series of hexadecimal numeric values in String like
syntax with `x` prefix.

Example: `x"616231"`

Tools generating CPON are discouraged from using this type.

### String
Represented as series of characters wrapped in `"`. It can contain any
characters except of the following ones that are mapped to these escape
sequences:

| Character            | Escape sequence |
|----------------------|-----------------|
| `\`                  | `\\`            |
| `"`                  | `\"`            |
| HT (horizontal tab)  | `\t`            |
| CR (carriage ret)    | `\r`            |
| LF (new line)        | `\n`            |
| FF (form feed)       | `\f`            |
| BS (backspace)       | `\b`            |
| NUL (null character) | `\0`            |

Octal, hexadecimal and Unicode escape sequences are not support (this is
incompatibility with JSON).

Example: `"some\tstring"`

### DateTime
Represented as **String** with `d` prefix where string's value is date, time and
optional time zone in ISO-8601 format (e.g. `d"2017-05-03T15:52:31.123"`)

### List
Represented as sequence of other RPC values wrapped in `[]`. Th sequence is
delimited by comma (`,`). The additional comma is allowed after last item (JSON
incompatible).

Examples: `[1 2 3]`, `[1,2,3,]`

### Map
Represented as pairs of **String** key and RPC value delimited by colon (`:`).
All pairs are wrapped in `{}` and delimited by comma (`,`). Contrary to JSON's
objects comma after last pair is allowed.

Examples: `{"one": 1, "dec": 1.22,}`

### IMap
Reprensented as pairs of **Int** key and RPC value delimited by colon (`:`). All
pairs are wrapped in `{}` and delimited by comma (`,`). Contrary to JSON's
object comma after past pair is allowed.

Examples: `{1: "one", 2: b"foo",}`

### MetaMap
Represented as pairs of **String** or **Int** key and RPC value delimited by
color (`:`). All pairs are wrapped in `<>` and delimited by comma (`,`). Comma
after past pair is allowed. **MetaMap** can't be alone, it must be followed by
some other RPC value (that is not **MetaMap**).

Example: `<1: "foo", "date": d"2017-05-03T15:52:31.123">42`


## Syntax highlighting support
Syntax definition for tree sitter can be found at
<https://github.com/amaanq/tree-sitter-cpon>
