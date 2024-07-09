# Implementations

This is the list of projects implementing at least ChainPack and/or CPON data
formats. Most of the provide the abstraction on SHV RPC.

## C

### [shvc](https://gitlab.com/elektroline-predator/shvc)

C implementation targeting Unix-like operating systems. This includes support
for embedded OS [NuttX](https://nuttx.apache.org/), and thus this implementation
is concerned with memory size more than others. It is using C23 with GNU
extensions.

This project also implements some tools that can be used from CLI and SHV RPC
Broker.

### [libshv](https://github.com/silicon-heaven/libshv/tree/master/libshvchainpack/c)

Implementation of ChainPack and CPON formats. The transport layers or any SHV
RPC APIs are not provided. The implementation is `malloc()` free and depends
only on C99.


## C++

### [libshv](https://github.com/silicon-heaven/libshv)

The most advanced implementation of SHV but also with highest historical baggage
relative to the previous SHV 3.0 versions.

The additional applications are provided based on this library in
[shvapp](https://github.com/silicon-heaven/shvapp) which include SHV RPC Broker.


## JavaScript

### [libshv-js](https://github.com/silicon-heaven/libshv-js)

Implementation intended to be used in web browsers to access SVH RPC over
WebSocket. It provides SHV RPC message exchange functionality, but higher levels
of SHV APIs integration is not provided.


## Python

### [pySHV](https://gitlab.com/elektroline-predator/pyshv)

Pure Python implementation intended as SHV 3.0 reference.

In combination with Python testing frameworks (such as
[pytest](https://pytest.org)) it is a very good option for testing any SHV
application.

This project also implements SHV RPC Broker and History recording as a
separately usable applications.


## Rust

### [libshvrpc-rs](https://github.com/silicon-heaven/libshvrpc-rs)

This project is split across multiple repositories and thus libraries providing
a way to select the minimal requirements for the specific application.

* [libshvproto-rs](https://github.com/silicon-heaven/libshvproto-rs) that
  provides ChainPack and CPON support.
* [libshvrpc-rs](https://github.com/silicon-heaven/libshvrpc-rs) providing basic
  support for SHV RPC based on libshvproto-rs.
* [libshvclient-rs](https://github.com/silicon-heaven/libshvclient-rs) is the
  ideal library if you intend to implement client application based on the
  libshvrpc-rs.
