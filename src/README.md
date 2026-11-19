# SHV RPC

SHV RPC is an RPC framework built around the ChainPack packing schema. An RPC
call is made by sending a [ChainPack](chainpack.md)-encoded
[Message](rpcmessage.md), either through a broker or peer-to-peer.

Why ChainPack? I liked XML attributes, but I dislike fact that XML attributes
syntax is not XML. I like JSON brevity, but you cannot have attributes and
comments. ChainPack is an attempt to choose the best from XML and JSON.

ChainPack's main features:
* UTF-8 string encoding
* every value can have a set of named attributes, whose values are ChainPack
  values as well
* binary `ChainPack` and text `Cpon` representations are mutually convertible;
  see the `cp2cp` or `ccp2cp` utilities
* `DateTime` native type
* `Blob` native type

See available implementations [for various programming
languages](implementations.md) and [various tools](tools.md).
