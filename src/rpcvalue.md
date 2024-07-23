# RPC Value

SHV RPC messages are encoded in JSON like meta-data format. There are two
defined representations: human-readable CPON and binary ChainPack.

- [ChainPack](./chainpack.md)
- [CPON](./cpon.md)

## Types

A single RPC Value has one of the following types while some of them are
containers and thus more complex representations can be constructed.

| Type         | Description                                                                                                                                      |
| ------------ | -----------------------------------------------------------------------------------------------------------------------------------------        |
| `Null`       | No value used to signal unset or even not-present state
| `Bool`       | Standard boolean with true or false value
| `Int`        | Signed integer can be up to 17 bytes long (most of the implementations cap on 8 bytes)
| `UInt`       | Unsigned integer that (can also by up to 17 bytes long)
| `Double`     | IEEE 754 double-precision binary floating-point
| `Decimal`    | Integer number with fixed decimal point
| `Blob`       | Sequence of bytes
| `String`     | UTF-8 encoded string
| `DateTime`   | Date and time with optional time zone
| `List`       | Ordered sequence of RPC Values
| `Map`        | Mapping from `String` keys to RPC Values
| `IMap`       | Mapping from `Int` keys to RPC Values
| `MetaMap`    | Mapping from `String` or `Int` keys to RPC Values. MetaMap must be associated with some other type and stores additional information for it.

Conversion of all data types between CPON and ChainPack is mostly without any
information lost with exception of exact encoding that is not possible to cary
over (such as String and CString in ChainPack converging to just String in CPON,
or CPON's HexBlob and Blob converting to ChainPack's Blob).
hexadecimal numbers 

## Some examples in Cpon

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
