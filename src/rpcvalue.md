# RpcValue

RpcValue is like JSON variable with optional meta-data appended.

## RpcValue types

Type | Description
-----|-----------
`Null` | 
`Bool` | 
`UInt` | 
`Int` | 
`Double` | IEEE 754 double-precision binary floating-point
`Decimal` | integer number with fixed decimal point
`Blob` |  binary string
`String` |  UTF8 encoded string
`DateTime` | 
`List` | list of rpc values 
`Map` | map of rpc values, keys are of `String` type
`IMap` | map of rpc values, keys are of `Int` type
`MetaMap` | map of rpc values, keys are of `Int` or `String` type, meta map is not basic type, it is used for storing values meta information only. 

RpcValue can hold all JSON types, but it defines also some new types like `DateTime` or `Imap`.

RpcValue can be represented in binary [ChainPack](chainpack.md) or text [Cpon](cpon.md) format.

Both formats are convertible each to the other without information loss. 
The only exception is `Double` value, which may not have always exact text representation.

## Some RpcValue examples in Cpon

Utf8 string with meta-information about its format.
```
<"format":"Date">"2023-01-02"	
```

ID Int.
```
<"type":"ID">123	
```

Addressbook entry
````
<1:"AdressBookEntry", "format":"cpon">
{
  "name": "John",
  "birth": <"format": "ISODate">"2000-12-11",
}
````