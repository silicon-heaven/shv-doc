# ChainPack

## PackedValue
```
+---------------+------------+
| PackingSchema | PackedData |
+---------------+------------+
```
* `PackingSchema` - uint8_t
* `PackedData` - binary blob of arbitrary length interpreted according to `PackingSchema`

## PackingSchema

Dec | Hex | Bin | Name
----|-----|-----|-----
128 | 80 | 10000000 | Null
129 | 81 | 10000001 | UInt
130 | 82 | 10000010 | Int
131 | 83 | 10000011 | Double
132 | 84 | 10000100 | Bool
133 | 85 | 10000101 | Blob
134 | 86 | 10000110 | String
136 | 88 | 10001000 | List
137 | 89 | 10001001 | Map
138 | 8a | 10001010 | IMap
139 | 8b | 10001011 | MetaMap
140 | 8c | 10001100 | Decimal
141 | 8d | 10001101 | DateTime
142 | 8e | 10001110 | CString 
143 | 8f | 10001111 | BlobPart (Experimental)
253 | fd | 11111101 | FALSE
254 | fe | 11111110 | TRUE
255 | ff | 11111111 | TERM

### TERM
Special type. Terminates packed List and Map elements
### FALSE
No packed data after *PackingSchema*. Optimization type, Used to save 1 byte when bool value is packed 
### TRUE
No packed data after *PackingSchema*. Optimization type, Used to save 1 byte when bool value is packed 
### Null
No packed data after *PackingSchema*. 
### UInt
Values 0-63 are packed directly in `PackingSchema` to save one byte.

LSB is the least significant byte
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
example:
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
### Int
Values 0-63 are packed directly in `PackingSchema` like `64+n` to save one byte.

Signed int is stored in the same way as the unsigned one, the only difference is that the sign bit is stored at position marked by `s`

LSB is the least significant byte

s is sign bit
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
example:
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
### Double 
64 bit blob, because this layout depends on system endianness, the *LITTLE ENDIAN* is used.
example:
```
0.0:       
-40000.0:  
```
### Decimal
```
+-------------+---------------+
| Int mantisa | Int exponent  |
+-------------+---------------+
```
Exponential number with exponent of base `10`. Base `2` might be used in future for type `BinaryExp`

value = mantissa * (base ^ exponent)

both `mantisa` and `exponent` can have value of `TERM` to encode special values

`exponent` == `TERM` :
* `mantisa == 1` - `+INF`
* `mantisa == -1` - `-INF`
* `mantisa == 0` - `qNaN`
* `mantisa == 2` - `sNaN`

`mantisa` == `TERM` : RESERVED

`INF` and `NaN` is not supported yet

### Bool
one byte, 0 - false, 1 - true

### String
```
+-------------+------------+
| UInt length | utf-8 data |
+-------------+------------+
```
example:
```
"fpowf":  10001010|00000101|01100110|01110000|01101111|01110111|01100110
```

### Blob, BlobPart
```
+-------------+------+
| UInt length | data |
+-------------+------+
```
example:
```
"fpowf\u0000sapofkpsaokfsa":  10001010|00010100|01100110|01110000|01101111|01110111|01100110|00000000|01110011|01100001|01110000|01101111|01100110|01101011|01110000|01110011|01100001|01101111|01101011|01100110|01110011|01100001
```
If embedded device has not buffer long enough to keep whole blob before sending, so it does not know also blob length, which must be sent first, then `BlobPart` message comes to play. 
`BlobPart` format is exactly same as the `Blob` one, but it keeps parser informed, that rest of overal blob will come in more parts. 
So every blob mesage can consist of zero or more `BlobPart` chunks plus exactly one `Blob` one, like `[BlobPart]*[Blob]`.

### CString
`CString` is stream of `utf-8` valid bytes terminated by `\0`. `CString` MUST NOT contain `\0` byte inside, so escaping is not needed.
```
+--------------+------+
| escaped data | `\0` |
+--------------+------+
```
example:
```
"fpowfu0000sapofkpsaokfsa":  10001110|01100110|01110000|01101111|01110111|01100110|00000000
```

### DateTime
```
bit 0: has TZ flag
bit 1: has not msec part flag
```
* Take Int value `msecs since 2018-02-02`
* if msec part == 0 `val /= 1000`
* if UTC offset != 0 `val = (val << 7) + (utc_offset_min / 15)` -63 <= offset <= 63
* `val <<= 2`
* set flags for TZ and msec part

note:

Current libshv implementation stores datetime msecs in:
* `signed 55 bit int` with TZ. This covers dates from year `-571232` to `571232`
* `signed 62 bit int` without TZ. This covers dates from year `-73117802` to `73117802`

example:
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
```
+------------+-----+------------+------+
| RpcValue_1 | ... | RpcValue_n | TERM |
+------------+-----+------------+------+
```
example:
```
["a",123,true,[1,2,3],null]
10001100|10001010|00000001|01100001|10000110|01000000|01111011|10000011|10001100|01000001|01000010|01000011|11111111|10000100|11111111
```
### Map
Key value container with unique keys, key order is not guaranteed.
```
+--------------+------------+-----+--------------+------------+------+
| String key_1 | RpcValue_1 | ... | String key_n | RpcValue_n | TERM |
+--------------+------------+-----+--------------+------------+------+
```
example:
```
{"bar":2,"baz":3,"foo":1}
10001101|00000011|01100010|01100001|01110010|01000010|00000011|01100010|01100001|01111010|01000011|00000011|01100110|01101111|01101111|01000001|11111111
{"bar":2,"baz":3,"foo":[11,12,13]}
10001101|00000011|01100010|01100001|01110010|01000010|00000011|01100010|01100001|01111010|01000011|00000011|01100110|01101111|01101111|10001100|01001011|01001100|01001101|11111111|11111111
```
### IMap
Key value container with unique keys, key order is not guaranteed.
```
+-----------+------------+-----+-----------+------------+------+
| Int key_1 | RpcValue_1 | ... | Int key_n | RpcValue_n | TERM |
+-----------+------------+-----+-----------+------------+------+
```
example:
```
i{1:"foo",2:"bar",333:15}
10001110|00000001|10001010|00000011|01100110|01101111|01101111|00000010|10001010|00000011|01100010|01100001|01110010|10000001|01001101|01001111|11111111
```
### MetaMap
Like `IMap` but can have also `String` keys. 
Key order is not guaranteed.

### RpcValue with MetaData
MetaData are prepended before packed RpcValue in form of dictionary quoted by `<` and `>`. MetaMap may contain Int and String keys.

