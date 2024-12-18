# Types descriptions

SHV RPC provides basic simple types such as integer or string but also
containers such as list or map. In combination that provides a way to have any
complex data structures. The hinting on expected parameter and result type
provides applications calling those methods with reasonable limits. In effect,
type descriptions allow some client-side data input validation for parameters
and advanced data interpretation for the results.

The format is chosen to be compact and machine readable but still readable and
writable by human.

The general rules are:

* The characters `[]{}():,|` are reserved and can't be used in `KEY` and `UNIT`.
* No white characters are allowed between types (the only exception is `UNIT`
  where they are interpreted as part of the unit text).
* Numerical constants are decimal (hex nor bin are supported) with optional
  leading `-`. The leading `+` is not allowed. Integer numbers can also be
  prefixed with either `^` or `>` (must be used after `-`). The `^` symbol can
  be used if you need power of two number (`-^8` is `-256`). The `>` synbol can
  be used if you need power of two minus one (`->8` is `-255`). These extensions
  are provided as these numbers are common numerical limits in computers.
* Some numerical constants are decimal point numbers. These do not support `^`
  and `>` prefixes and use decimal point (`.`, not comma). They can start with
  `.` (the leading zero can be left out).


## Null

```
n
```

This type means that only *Null* can be used. This is also understood as "not
provided" in various situations (such as the whole parameter, and item in the
*Map*, *iMap*, and *List*).

## Bool

```
b
```

The *Bool* type. There are no special additional modifiers you can specify here.

## Integer

```
i           iUNIT
i(MIN,MAX)  i(MIN,MAX)UNIT
```

The *Int* type that can optionally have range specified as numerical constants.
Any of the limits can also be unspecified and in such case it is not applied.

Type specifier can be followed by unit specification (`UNIT`).

Examples:

* `i(0,)` Integer that can be any positive number or zero but not negative.
* `i(128,255)` Integer in range from 128 to 255 (including those numbers).
* `i(^7,>8)` This is same as the previous example but uses power of two
  specification.
* `i°C` Integer of any value that has unit specified.

## Unsigned integer

```
u           uUNIT
u(MAX)      u(MAX)UNIT
u(MIN,MAX)  u(MIN,MAX)UNIT
```

The *UInt* type that can optionally have range or maximal value specified as
numerical constant (but only positive numbers are accepted). Any of the limits
can also be unspecified and in such case it is not applied.

Type specifier can be followed by unit specification (`UNIT`).

⚠️ The usage of unsigned integers is discouraged because it introduces simple to
miss type mismatches and in most cases can be safely replaced by integer with
an appropriate range.

## Enum

```
i[KEY,KEY:INDEX,...]
```

This is *Int* that can have only predefined values. For every value this type
also provides alias (`KEY`). The values are assigned in sequence from zero. You
can also set any number to the alias with `KEY:INDEX` syntax. In such case the
subsequent `KEY` will have `INDEX+1` assigned and so on. Be aware that having
more than one alias for single index is forbidden.

Examples:

* `i[TRUE,FALSE,INVALID]` Enum where `0` is aliased with `TRUE`, `1` with
  `FALSE` and `2` with `INVALID`.
* `i[fail:-1,success]` Enum where `-1` is aliased with `fail` and `0` with
  `success`.

## Double (floating point number)

```
f  fUNIT
```

The *Double* type. It can be followed by unit specification (`UNIT`).

Examples:

* `f%` Floating point in percents.

## Decimal

```
d                      dUNIT
d(MIN,MAX)             d(MIN,MAX)UNIT
d(MIN,MAX,PRECISSION)  d(MIN,MAX,PRECISSION)UNIT
```

The *Decimal* type that can optionally have range or range and precision
specified as decimal point numerical constants. The precision is number of
decimal points expected. The negative number can be also used for the
precission. Any of the limits can be unspecified and in such case it is not
applied.

Type specifier can be followed by unit specification (`UNIT`).

Examples:

* `d(0.3,0.8)` Decimal that must be in range of <0.3, 0.8>.
* `d(0,100,2)%` Decimal in percents that can be only from zero to 100 with
  precision of `0.01`. The unit is `%`.
* `d(1000,2000,-2)` Decimal that can be from one thousands to two thousands in
  steps of hundred.
* `d(,,2)` Decimal that can be any number but only with precission of `0.01`.

## String

```
s
s(LEN)
s(MIN,MAX)
```

The *String* type with optional length limitation specified as numerical
constants. One of the limits can be left out and in such case it is not applied.

Examples:

* `s(0,63)` String with maximal length 63 characters.
* `s(16)` String with exactly 16 characters.

## Blob

```
x
x(LEN)
x(MIN,MAX)
```

The *Blob* type with optional length limitation specified as numerical
constants. One of the limits can be left out and in such case it is not applied.

Examples:

* `x(0,42)` Blob with maximal length of 42 bytes.
* `x(1)` Blob with exactly 1 byte.

## DateTime

```
t
```

The *DateTime* type.

## List

```
[TYPE]
[TYPE](LEN)
[TYPE](MIN,MAX)
```

The *List* type. This list can have only items of specified type (`TYPE`) and
number of them can be limited. The limits can be unspecified and in such case
they are not applied.

Examples:

* `[i(0,100)](2)` List of integers between 0 and 100 with exactly two items.
* `[?](1,4)` List that can contain anything as far as it has from 1 to 4 items.
* `[s](1,)` List that can contain strings with minimum of at least one.

## Tuple

```
[TYPE:KEY,...]
```

The *List* type with fixed length and defined types per its items. You can
specify any number of types in the list (but at least one must be specified).
The `KEY` is the alias for the item.

Any value that allows *Null* (such as `i|n`) can be left out if all further
items are *Null* as well. For example `[i|n:foo,d|n:faa]` accepts `[42,1.8]` as
well as `[42]` or `[]` or even `[null,1.8]`.

Notice that syntax is pretty much the same as for *List* type in the previous
section. The difference is that for *Tuple* the `KEY` must be provided and
multiple items can be specified.

Examples:

* `[i:id,s:name,t|n:lastLogin]` First item must be integer, second item string
  and third is optional and if present must contain date and time. Thus list can
  have two or three fields in this case.

## IMap

```
i{TYPE}
```

The *IMap* type. This IMap can have only values of specified type (`TYPE`).

Examples:

* `i{s}` IMap where values are strings.

## Struct

```
i{TYPE:KEY,TYPE:KEY:IKEY,...}
```

The *IMap* type that has predefined items. Every item must have an unique
string identifier (`KEY`) and integer identifiers are incrementally assigned
from 0. You can use `TYPE:KEY:IKEY` variant to specify integer identifier for
the item and subsequent `TYPE:KEY` item will have `IKEY+1` assigned and so on.

The integer identifier is used in data to identify item and string identifier
can be used by applications as an alias when interpreted for the user.

Any item that allow type *Null* (such as `i|n`) can be left out completely in
the data.

Notice that syntax is close to the *IMap* type in the previous section. The
difference is that for *IMap* only type is specified while for *Struct* KEY must
be specified as well.

Examples:

* `i{d:date,i(0,63):level,s:id,?:info}` The description of standard
  alert as defined in [device API](./rpcmethods/device.md#devicealertsget).
* `i{s:name:1,u[,isGetter,isSetter,largeResult,notIndempotent,userIDRequired]|n:flags,s|n:paramType,s|n:resultType,i(0,63):accessLevel,{s|n}:signals,{?}:extra:63}`
  The method info as defined in [discovery API](./rpcmethods/discovery.md#dir).

## Map

```
{TYPE}
```

The *Map* type. This Map can have values only of the specified type (`TYPE`).

Examples:

* `{i}` Map where values are integers.

## KeyStruct

```
{TYPE:KEY,...}
```

The *Map* type that has predefined items. Every item must have an unique string
identifier (`KEY`).

Any item that allow type *Null* (such as `i|n`) can be left out completely in
the data.

Notice that syntax is close to the *Map* type in the previous section. The
difference is that for *Map* only type is specified while for *KeyStruct* KEY
must be specified as well.

⚠️ Usage of KeyStruct is discouraged because it can be replaced by Struct that in
the transfer has smaller size and provides the same functionality if combined
with its type description.

## Bitfield

```
u[TYPE:KEY,TYPE:KEY:INDEX,...]
```

The *UInt* that has other types encoded. This is kind of like Tuple but it
allows much smaller data encoding with exception that not every type can be
included. The only allowed types are:

* Boolean (`b`): uses single bit where `0` is false and `1` is true
* Unsigned integer with maximum specified (`u(MAX)`): uses maximum number of
  bits required to store `MAX`.
* Unsigned integer with minimum and maximum specified (`u(MIN,MAX)`): uses
  maximum number of bits required to store the `MAX-MIN`. This values stored in
  the bitfield are shifted by `MIN`.
* Enum but only if no negative value is defined (`i[...]`): uses number of bits
  required to store the largest integer value.

Every type spans predefined number of bits in the integer starting with least
significant one. Bits are indexed from zero and thus you can also see it as
counting used bits from `0`. The `TYPE:KEY:INDEX` variant allows you to specify
a specific offset and this it allows creating uninterpreted holes. The
subsequent `TYPE:KEY` are aligned after the last used bit. It is not allowed to
reference same bit in two different type specifier (with an appropriate
`INDEX`).

⚠️ It is highly suggested not to use more than 64 bits as that is common
implementation limit for unsigned integers.

Examples:

* `u[i[OK,STARTUP,ERROR]:status,b:debug]` The bitfield where two least
  significant bits are used to store value for enum (`0` is `OK`, `1` is
  `STARTUP` and `2` is `ERROR`), and the third bit is used to store boolean
  `debug`. In total this is an unsigned integer with largest possible value 8
  (three bits used).
* `u[u(32):phase,u(24,32):outOf]` This is bitfield where least significant five
  bits are used to store unsigned integer with value up to 32 and subsequent
  three bits are used to store integer between 24 and 32. The total unsigned
  integer size is 8 bits.

## One of types

```
TYPE|TYPE
```

This can be used to combine multiple types where type is expected. It provides
"or" functionality: "must be one of the specified type".

Examples:

* `i|n` Either integer or null is allowed.
* `i|d|s` Integer, decimal and string are allowed.
* `i(-10,-5)|i(5,10)` Integer in range -10 to -5 or 5 to 10

## Any type

```
?
?(ALIAS)
```

This is type that allows any valid SHV type. It is provided to allow generic
data structures to be included in the type description.

The `?(ALIAS)` variant is provided to allow implementation to refer to some
external type reference with given `ALIAS` that can be anything that doesn't
include `)` character (as that is termination character).

## Standard types

This standard defines few of the different complex structures and to signal
their type aliases are provided. This reduces size of the standard methods
descriptions.

* `!dir` is one of results of [`*:dir` method](./rpcmethods/discovery.md#dir).
  Its expanded form is:
  ```
  i{s:name:1,u[b:isGetter:1,b:isSetter,b:largeResult,b:notIndempotent,b:userIDRequired]|n:flags,s|n:paramType,s|n:resultType,i(0,63):accessLevel,{s|n}:signals,{?}:extra:63}|b
  ```
* `!alert` is result of [`.device/alerts:get`
  method](./rpcmethods/device.md#devicealertsget) and value of
  [`.device/alerts:get:chng`
  signal](./rpcmethods/device.md#devicealertsgetchng).
  ```
  i{t:date,i(0,63):level,s:id,?:info}
  ```
* `!stat` is result of [`*:stat` method](./rpcmethods/file.md#stat). Its
  expanded form is:
  ```
  i{i:type,i:size,i:pageSize,t|n:accessTime,t|n:modTime,i|n:maxWrite}
  ```
* `!exchangeP` is parameter for [`*/ASSIGNED:exchange`
  method](./rpcmethods/exchange.md#ASSIGNEDexchange). Its expanded form is:
  ```
  i{u:counter,u|n:readyToReceive,b|n:data:3}
  ```
* `!exchangeR` is result of [`*/ASSIGNED:exchange`
  method](./rpcmethods/exchange.md#ASSIGNEDexchange). Its expanded form is:
  ```
  i{u|n:readyToReceive:1,u|n:readyToSend,b|n:data}
  ```
* `!exchangeV` is value of
  [`*/ASSIGNED:exchange:ready`
  signal](./rpcmethods/exchange.md#ASSIGNEDexchangeready). Its expanded form is:
  ```
  i{u|n:readyToReceive:1,u|n:readyToSend}
  ```
* `!getLogP` is parameter for
  [`.history/**:getLog` method](./rpcmethods/history.md#historygetlog). Its
  expanded form is:
  ```
  {t|n:since,t|n:until,i(0,)|n:count,b|n:snapshot,s|n:ri}
  ```
* `!getLogR` is result of
  [`.history/**:getLog` method](./rpcmethods/history.md#historygetlog). Its
  expanded form is:
  ```
  [i{t:timestamp:1,i(0,)|n:ref,s|n:path,s|n:signal,s|n:source,?:value,s|n:userId,b|n:repeat}]
  ```
* `!historyRecords` is result of
  [`.history/**/.records/*:fetch`
  method](./rpcmethods/history.md#historyrecordsfetch). Its expanded form is:
  ```
  [i{i[normal:1,keep,timeJump,timeAbig]:type,t:timestamp,s|n:path,s|n:signal,s|n:source,?:value,i(0,63):accessLevel,s|n:userId,b|n:repeat,i|n:timeJump:60}]
  ```

## Grammar representation

```
type             = bool | integer | unsigned integer | enum | double | decimal | string | blob | date time | list | tuple | imap | struct | map | keystruct | bitfield | one of | standard;

bool             = "b";
integer          = "i", ["(", [number], ",", [number], ")"], [unit];
unsigned integer = "u", ["(", [positive number], ",", [positive number], ")"], [unit];
enum             = "i", "[", enum item, {",", enum item}, "]"
double           = "f", [unit];
decimal          = "d", ["(", [decimal number], ",", [decimal number], [",", decimal number], ")"], [unit];
string           = "s", ["(", [number], [",", number], ")"];
blob             = "b", ["(", [number], [",", number], ")"];
date time        = "t";
list             = "[", type, "]", ["(", [number], [",", [number]], ")"];
tuple            = "[", key item, {",", key item}, "]";
imap             = "i", "{", type, "}";
struct           = "i", "{", ikey item, {",", ikey item}, "}";
map              = "{", type, "}";
keystruct        = "{", key item, {",", key item}, "}";
bitfield         = "u", "[", ikey item, {",", ikey item}, "]";
one of           = type - oneof, "|", type;
standard         = "!", text;

unit             = text;
enum item        = text, [":", number];
key item         = type, ":", text;
ikey item        = type, ":", text, [":", number];
decimal number   = ["-"], ({digit} | [digit] , [".", {digit}]);
number           = "0" | ["^" | "<"], ["-"], digit - "0", {digit};
positive number  = "0" | ["^" | "<"], digit - "0", {digit};
digit            = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";
text             = character, {character};
character        = ? UTF-8 character ? - ("[" | "]" | "{" | "}" | "(" | ")" | ":" | "," | "|");
```
