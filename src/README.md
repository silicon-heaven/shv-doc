# SHV RPC

SHV RPC is RPC framework build around ChainPack packing schema.
RPC call is realized as sending [ChainPack](chainpack.md) encoded [RpcMessage](rpcmessage.md) either via broker or peer-to-peer.

Why ChainPack? I liked XML attributes, but I dislike fact that XML attributes syntax is not XML. 
I like JSON brevity, but you cannot have attributes, comments, utf8 in JSON. ChainPack is
attempt to choose the best from XML and JSON.

ChainPack main features:
* UTF8 string encoding
* every value can have set of named attributes, attributes values are ChainPack again.
* binary `ChainPack` and text `Cpon` representation convertible each to other, see `cp2cp` or `ccp2cp` utilities
* `DateTime` native type
* `Blob` native type

Reference implementation is written in C and C++ <https://github.com/silicon-heaven/libshv> 

There is also pure python SHV [implementation](https://gitlab.com/elektroline-predator/pyshv) 
with great [documentation](https://elektroline-predator.gitlab.io/pyshv/master/index.html) thanks to [Karel Kočí](https://gitlab.com/Cynerd). 
This book contains some parts of Karel's original document.

There are also other [language bindings](language-bindings.md) available.