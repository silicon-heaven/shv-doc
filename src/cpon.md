# Cpon - ChainPack Object Notation

Superset of JSON with following extensions:
* **String** - C-escaped Utf8 encoded string, can contain any char except of `"`, `\`, `TAB`, `CR`, `LF`, `0`. Supported escape sequences are `\0`, `\b`, `f`, `\n`, `\r`, `\t`. Octal, hexadecimal and unicode `\xHH` and `\uHHHH` sequences are not supported yet.
* **Blob**
  * **Escaped** - `b"..."` values `0x00 - 0x1f` and `0x7f - 0xff` are stored as escape sequence `\hh`, `\` as `\\`, `"` as `\"`, `TAB` as `\t`, `CR` as `\r`, `LF` as `\n`, other values as ASCII `b"ab\31"` is the same as `b"ab1"`
  * **Hex** - `x"..."` char values are saved as hh, for example `"ab1"` is coded as `x"616231"` 
* **MetaMap** - `< int_key: val1 , "string_key": val2, ...  >` with Int or String keys, example `<3:42u, "foo":"bar">`
* **IMap** - `i{ key: val , ... }`, map with int keys, example `i{3:42u, 4:"abc"}`
* **hexadecimal** notation for int is supported, example: `0x20`, `0x40u`
* **UInt** - u suffix, example `123u` is `UInt(123)`
* **Decimal** - floating point numbers with decimal point `.` or in exponential format, example: `123.45`, `12345e-2`. Decimals are not converted to double, the are stored as mantisa + exponent instead without loose of precision. For example 1.1 is stored as pair (11,-1) 
* **DateTime** - d prefix, example `d"2017-05-03T15:52:31.123"`
* **List** and Map fields can be delimited by any white-space character including new line, ie. `[1 2 3]` means `[1,2,3]`
* **List** and Map fields can have delimiter `,` after last item like: `[1,2,3,]`
* C style comments are supported, for example `[1,/*2,*/3]` means `[1,3]`
 
Example:
```
<"type": "temperature">32
```
```
<1:"AdressBookEntry">
{
  "name": "John",
  "birth": <"format": "ISODate">"2000-12-11",
}
```
[Tree sitter syntax definition](https://github.com/amaanq/tree-sitter-cpon)