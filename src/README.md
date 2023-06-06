# SHV RPC

SHV RPC is just another RPC framework build around ChainPack packing schema.

Why ChainPack? I liked XML attributes, but I dislike fact that XML attributes syntax is not XML. 
I like JSON brevity, but you cannot have attributes, comments, utf8 in JSON. ChainPack is
just another attempt to choose the best from XML and JSON.

ChainPack main features:
* UTF8 string encoding
* every value can have set of named attributes, attributes values are ChainPack again.
* text `ChainPack` and binary `Cpon` representation convertible each to other, see `cp2cp` or `ccp2cp` utilities
* `DateTime` native type
* `Blob` native type

SHV RPC is just about sending `ChainPack` encoded RPC messages either via broker or peer-to-peer.
