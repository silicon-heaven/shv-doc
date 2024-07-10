# SHV RPC

SHV RPC is RPC framework build around ChainPack packing schema. RPC call is
realized as sending [ChainPack](chainpack.md) encoded [Message](rpcmessage.md)
either via broker or peer-to-peer.

Why ChainPack? I liked XML attributes, but I dislike fact that XML attributes
syntax is not XML. I like JSON brevity, but you cannot have attributes,
comments, utf8 in JSON. ChainPack is attempt to choose the best from XML and
JSON.

ChainPack main features:
* UTF8 string encoding
* every value can have set of named attributes, attributes values are ChainPack
  again.
* binary `ChainPack` and text `Cpon` representation convertible each to other,
  see `cp2cp` or `ccp2cp` utilities
* `DateTime` native type
* `Blob` native type

See available implementations [for various programming
languages](implementations.md) and [various tools](tools.md).
