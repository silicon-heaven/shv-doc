# ChainPack

ChainPack is binary data format that caries type information with data. It is
designed to be read and written in streams.

Note that different implementations can have different limitations on data
representation compared to the full range ChainPack supports. Always consult
your implementation's documentation to identify them.

Every value is prefixed with a single byte that specifies format of the
following data, referenced as *packing schema* in this document. The following
formats are defined:

| Dec | Hex | Bin      | Name      |
|-----|-----|----------|-----------|
| 128 | 80  | 10000000 | Null      |
| 129 | 81  | 10000001 | UInt      |
| 130 | 82  | 10000010 | Int       |
| 131 | 83  | 10000011 | Double    |
| 133 | 85  | 10000101 | Blob      |
| 134 | 86  | 10000110 | String    |
| 136 | 88  | 10001000 | List      |
| 137 | 89  | 10001001 | Map       |
| 138 | 8a  | 10001010 | IMap      |
| 139 | 8b  | 10001011 | MetaMap   |
| 140 | 8c  | 10001100 | Decimal   |
| 141 | 8d  | 10001101 | DateTime  |
| 142 | 8e  | 10001110 | Unused    |
| 143 | 8f  | 10001111 | BlobChain |
| 253 | fd  | 11111101 | FALSE     |
| 254 | fe  | 11111110 | TRUE      |
| 255 | ff  | 11111111 | TERM      |

The values from 0 to 127 are used as **UInt** and **Int** values and are more
discussed in their paragraphs.

### Null
Represented just with its *packing schema*, no additional bytes are expected.

```
+------+
| 0x80 |
+------+
```

### Bool
Represented with either **TRUE** or **FALSE** *packing schema*. There are no
additional bytes expected after it.

True:
```
+------+
| 0xfd |
+------+
```

False:
```
+------+
| 0xfe |
+------+
```

### Int
Signed integers can be packed directly in *packing schema* for values from 0 to
63 or with *packing schema* **Int** followed by bytes with integer value encoded
in a specific way.

The values from 0 to 63 can be packed directly in *packing schema* by adding 64
to them. The value 42 thus would be packed as `0x6a`.

```
+--------------+
| 0x40 + Value |
+--------------+
Value ∈ <0, 63>
```

```
+------+---------+
| 0x82 | Data ...
+------+---------+
```

Signed integers are stored in their absolute value with sign bit. Number of
bytes integer spans can be decoded from the first byte in the way described in
the following visualization.

Bytes in stream must be from left to right and unsigned integer itself is
included with its most significant bit in first byte that should carry it
(big-endian).

```
 0 ...  6 bits  1  byte  |0|s|x|x|x|x|x|x|<-- LSB
 7 ... 13 bits  2  bytes |1|0|s|x|x|x|x|x| |x|x|x|x|x|x|x|x|<-- LSB
14 ... 20 bits  3  bytes |1|1|0|s|x|x|x|x| |x|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x|<-- LSB
21 ... 27 bits  4  bytes |1|1|1|0|s|x|x|x| |x|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x|<-- LSB
28+       bits  5+ bytes |1|1|1|1|n|n|n|n| |s|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x| ... <-- LSB
                    n ==  0 ->  4 bytes number (32 bit number)
                    n ==  1 ->  5 bytes number
                    n == 13 -> 17 bytes number
                    n == 14 -> RESERVED
                    n == 15 -> not used because it is TERM type
```
`s` is sign bit and `x` is big-endian unsigned integer bits.

The representation must strictly be sent in lowest possible bytes length.

Examples:
```
             4            0x4 ... len:  1  dump:  01000100
            16           0x10 ... len:  1  dump:  01010000
            64           0x40 ... len:  3  dump:  10000010|10000000|01000000
          1024          0x400 ... len:  3  dump:  10000010|10000100|00000000
          4096         0x1000 ... len:  3  dump:  10000010|10010000|00000000
         16384         0x4000 ... len:  4  dump:  10000010|11000000|01000000|00000000
        262144        0x40000 ... len:  4  dump:  10000010|11000100|00000000|00000000
       1048576       0x100000 ... len:  5  dump:  10000010|11100000|00010000|00000000|00000000
       4194304       0x400000 ... len:  5  dump:  10000010|11100000|01000000|00000000|00000000
      67108864      0x4000000 ... len:  5  dump:  10000010|11100100|00000000|00000000|00000000
     268435456     0x10000000 ... len:  6  dump:  10000010|11110000|00010000|00000000|00000000|00000000
    1073741824     0x40000000 ... len:  6  dump:  10000010|11110000|01000000|00000000|00000000|00000000
   17179869184    0x400000000 ... len:  7  dump:  10000010|11110001|00000100|00000000|00000000|00000000|00000000
   68719476736   0x1000000000 ... len:  7  dump:  10000010|11110001|00010000|00000000|00000000|00000000|00000000
  274877906944   0x4000000000 ... len:  7  dump:  10000010|11110001|01000000|00000000|00000000|00000000|00000000
 4398046511104  0x40000000000 ... len:  8  dump:  10000010|11110010|00000100|00000000|00000000|00000000|00000000|00000000
17592186044416 0x100000000000 ... len:  8  dump:  10000010|11110010|00010000|00000000|00000000|00000000|00000000|00000000
70368744177664 0x400000000000 ... len:  8  dump:  10000010|11110010|01000000|00000000|00000000|00000000|00000000|00000000
     -4 ... len:  2  dump:  10000010|01000100
    -16 ... len:  2  dump:  10000010|01010000
    -64 ... len:  3  dump:  10000010|10100000|01000000
  -1024 ... len:  3  dump:  10000010|10100100|00000000
  -4096 ... len:  3  dump:  10000010|10110000|00000000
 -16384 ... len:  4  dump:  10000010|11010000|01000000|00000000
-262144 ... len:  4  dump:  10000010|11010100|00000000|00000000
```

### UInt
Unsigned integers can be packed directly in *packing schema* for values from 0
to 63 or with *packing schema* **UInt** followed by bytes with integer value.

Values packed directly to *packing schema* are packed as they are. The value 42
thus would be packed as `0x2a`.

```
+--------------+
| 0x00 + Value |
+--------------+
Value ∈ <0, 63>
```

```
+------+---------+
| 0x81 | Data ...
+------+---------+
```

Number of bytes integer spans can be decoded from the first byte in the way
described in the following visualization. Bytes in stream must be from left to
right and integer itself is included with its most significant bit in first byte
that should carry it (big-endian).

```
 0 ...  7 bits  1  byte  |0|x|x|x|x|x|x|x|<-- LSB
 8 ... 14 bits  2  bytes |1|0|x|x|x|x|x|x| |x|x|x|x|x|x|x|x|<-- LSB
15 ... 21 bits  3  bytes |1|1|0|x|x|x|x|x| |x|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x|<-- LSB
22 ... 28 bits  4  bytes |1|1|1|0|x|x|x|x| |x|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x|<-- LSB
29+       bits  5+ bytes |1|1|1|1|n|n|n|n| |x|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x| |x|x|x|x|x|x|x|x| ... <-- LSB
                    n ==  0 ->  4 bytes number (32 bit number)
                    n ==  1 ->  5 bytes number
                    n == 13 -> 17 bytes number
                    n == 14 -> RESERVED
                    n == 15 -> not used because it is TERM type
```
`s` is sign bit and `x` is big-endian unsigned integer bits.

The representation must strictly be sent in lowest possible bytes length.

Examples:
```
               2u              0x2 ... len:  1  dump:  00000010
              16u             0x10 ... len:  1  dump:  00010000
             127u             0x7f ... len:  2  dump:  10000001|01111111
             128u             0x80 ... len:  3  dump:  10000001|10000000|10000000
             512u            0x200 ... len:  3  dump:  10000001|10000010|00000000
            4096u           0x1000 ... len:  3  dump:  10000001|10010000|00000000
           32768u           0x8000 ... len:  4  dump:  10000001|11000000|10000000|00000000
         1048576u         0x100000 ... len:  4  dump:  10000001|11010000|00000000|00000000
         8388608u         0x800000 ... len:  5  dump:  10000001|11100000|10000000|00000000|00000000
        33554432u        0x2000000 ... len:  5  dump:  10000001|11100010|00000000|00000000|00000000
       268435456u       0x10000000 ... len:  6  dump:  10000001|11110000|00010000|00000000|00000000|00000000
     68719476736u     0x1000000000 ... len:  7  dump:  10000001|11110001|00010000|00000000|00000000|00000000|00000000
  17592186044416u   0x100000000000 ... len:  8  dump:  10000001|11110010|00010000|00000000|00000000|00000000|00000000|00000000
 140737488355328u   0x800000000000 ... len:  8  dump:  10000001|11110010|10000000|00000000|00000000|00000000|00000000|00000000
4503599627370496u 0x10000000000000 ... len:  9  dump:  10000001|11110011|00010000|00000000|00000000|00000000|00000000|00000000|00000000
```

### Double 
Double-precision floating-point as defined in IEEE 754-2008 in little-endian.

```
+------+-------+-------+-------+-------+-------+-------+-------+-------+
| 0x83 | Byte0 | Byte1 | Byte2 | Byte3 | Byte4 | Byte5 | Byte6 | Byte7 |
+------+-------+-------+-------+-------+-------+-------+-------+-------+
```

### Decimal
Exponential number of base 10 is packed as two **Int** types (without *packing
schema*) where first is the mantisa and second is exponent (`mantisa *
10^exponent`). The second **Int** (exponent) can also be special value `0xff`
(also use as **TERM**) to encode special values.

```
+------+---       ---+---         ---+
| 0x8c | Int mantisa | Int exponent |
+------+---       ---+---         ---+
```

The special values when exponent is `0xff`:
* `mantisa == 1` is positive infinity (`+INF`)
* `mantisa == -1` is negative infinity (`-INF`)
* `matinsa == 0` is quiet NaN (`qNaN`)
* `mantisa == 2` is signaling NaN (`sNaN`)
* Other values of mantisa are reserved


### Blob
Blob is sent with **UInt** (without *packing schema*) prefixed that specifies
number of data bytes. The data should be sent in little-endian.

```
+------+---       ---+---      ---+
| 0x85 | UInt length | Data bytes |
+------+---       ---+---      ---+
```

Example:
```
"fpowf\u0000sapofkpsaokfsa":  10001010|00010100|01100110|01110000|01101111|01110111|01100110|00000000|01110011|01100001|01110000|01101111|01100110|01101011|01110000|01110011|01100001|01101111|01101011|01100110|01110011|01100001
```

### BlobChain
Blob packed in parts. It provides a way to pack binary data that are not of
known size upfront. Data are sent in blocks with their **UInt** (without
*packing schema*) size prefixed and termination is done by packing zero as data
length.

```
+------+---       ---+---      ---+-- --+---+
| 0x8f | UInt length | Data bytes | ... | 0 |
+------+---       ---+---      ---+-- --+---+
```

### String
UTF-8 encoded string. It is sent with **UInt** (without *packing schema*)
prefixed that specifies number of data bytes (not characters!). The data must be
sent in little-endian.

```
+------+-------------+------------+
| 0x86 | UInt length | UTF-8 data |
+------+-------------+------------+
```

Example:
```
"fpowf":  10001010|00000101|01100110|01110000|01101111|01110111|01100110
```

### DateTime
Date and time encoded as **Int** (without *packing schema*) with these
conversion sequence:

* Take integer value `msecs since 2018-02-02`
* if msec part == 0 `val /= 1000`
* if UTC offset != 0 `val = (val << 7) + (utc_offset_min / 15)` -63 <= offset <= 63
* `val <<= 2`
* set flags for TZ and msec part

```
bit 0: has TZ flag
bit 1: has not msec part flag
```

Example:
```
d"2018-02-02 0:00:00.001"       len:  2  dump:  10001101|00000100
d"2018-02-02 01:00:00.001+01"   len:  3  dump:  10001101|10000010|00010001
d"2018-12-02 0:00:00"           len:  5  dump:  10001101|11100110|00111101|11011010|00000010
d"2018-01-01 0:00:00"           len:  5  dump:  10001101|11101000|10101000|10111111|11111110
d"2019-01-01 0:00:00"           len:  5  dump:  10001101|11100110|11011100|00001110|00000010
d"2020-01-01 0:00:00"           len:  6  dump:  10001101|11110000|00001110|01100000|11011100|00000010
d"2021-01-01 0:00:00"           len:  6  dump:  10001101|11110000|00010101|11101010|11110000|00000010
d"2031-01-01 0:00:00"           len:  6  dump:  10001101|11110000|01100001|00100101|10001000|00000010
d"2041-01-01 0:00:00"           len:  7  dump:  10001101|11110001|00000000|10101100|01100101|01100110|00000010
d"2041-03-04 0:00:00-1015"      len:  7  dump:  10001101|11110001|01010110|11010111|01001101|01001001|01011111
d"2041-03-04 0:00:00.123-1015"  len:  9  dump:  10001101|11110011|00000001|01010011|00111001|00000101|11100010|00110111|01011101
d"1970-01-01 0:00:00"           len:  7  dump:  10001101|11110001|10000001|01101001|11001110|10100111|11111110
d"2017-05-03 5:52:03"           len:  5  dump:  10001101|11101101|10101000|11100111|11110010
d"2017-05-03T15:52:03.923Z"     len:  7  dump:  10001101|11110001|10010110|00010011|00110100|10111110|10110100
d"2017-05-03T15:52:31.123+10"   len:  8  dump:  10001101|11110010|10001011|00001101|11100100|00101100|11011001|01011111
d"2017-05-03T15:52:03Z"         len:  5  dump:  10001101|11101101|10100110|10110101|01110010
d"2017-05-03T15:52:03.000-0130" len:  7  dump:  10001101|11110001|10000010|11010011|00110000|10001000|00010101
d"2017-05-03T15:52:03.923+00"   len:  7  dump:  10001101|11110001|10010110|00010011|00110100|10111110|10110100
```

### List
This is sequence of other RPC values. It starts with *packing schema* and is
terminate with **TERM** (`0xff`).

```
+------+---   ---+-- --+---   ---+------+
| 0x88 | Value 1 | ... | Value n | 0xff |
+------+---   ---+-- --+---   ---+------+
```

Example:
```
["a",123,true,[1,2,3],null]
10001100|10001010|00000001|01100001|10000110|01000000|01111011|10000011|10001100|01000001|01000010|01000011|11111111|10000100|11111111
```

### Map
Encoded as sequence of **String** (with *packing schema*) key and RPC value
pairs. The order of pairs is not guaranteed and should not be relied upon. The
last pair must be followed by **TERM** (`0xff`) that terminates the **Map**.

```
+------+---        ---+---   ---+-- --+---        ---+---   ---+------+
| 0x89 | String key 1 | Value 1 | ... | String key n | Value n | 0xff |
+------+---        ---+---   ---+-- --+---        ---+---   ---+------+
```

Example:
```
{"bar":2,"baz":3,"foo":1}
10001101|00000011|01100010|01100001|01110010|01000010|00000011|01100010|01100001|01111010|01000011|00000011|01100110|01101111|01101111|01000001|11111111
{"bar":2,"baz":3,"foo":[11,12,13]}
10001101|00000011|01100010|01100001|01110010|01000010|00000011|01100010|01100001|01111010|01000011|00000011|01100110|01101111|01101111|10001100|01001011|01001100|01001101|11111111|11111111
```

### IMap
Encoded as sequence of **Int** (with *packing schema*) key and RPC value pairs.
The order of pairs is not guaranteed and should not be relied upon. The last
pair must be followed by **TERM** (`0xff`) that terminates the **IMap**.

```
+------+---     ---+---   ---+-- --+---     ---+---   ---+------+
| 0x8a | Int key 1 | Value 1 | ... | Int key n | Value n | 0xff |
+------+---     ---+---   ---+-- --+---     ---+---   ---+------+
```

Example:
```
i{1:"foo",2:"bar",333:15}
10001110|00000001|10001010|00000011|01100110|01101111|01101111|00000010|10001010|00000011|01100010|01100001|01110010|10000001|01001101|01001111|11111111
```

Notice that keys are of type **Int** not **UInt**!

### MetaMap
Encoded like **Map** and **IMap** but with keys being both **Int** and
**String**.

```
+------+---               ---+---   ---+-- --+---               ---+---   ---+------+
| 0x8b | Int or String key 1 | Value 1 | ... | Int or String key n | Value n | 0xff |
+------+---               ---+---   ---+-- --+---               ---+---   ---+------+
```
