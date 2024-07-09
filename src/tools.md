# Tools

These are tools and utilities to be used with SHV RPC.


## Clients

These are clients for generic access of the SHV RPC. They provide an interactive
interface to discover nodes and methods as well as to interact with them.

* [shvspy](https://github.com/silicon-heaven/shvspy): Qt based graphical
  client
* [shvcli](https://github.com/silicon-heaven/shvcli): command line interface
  client


## Brokers

These are standalone server applications implementing SHV RPC Broker
functionality.

* [shvapp](https://github.com/silicon-heaven/shvapp): provides `shvbroker`
* [pyshv](https://github.com/silicon-heaven/pyshv): provides `pyshvbroker`
* [shvbroker-rs](https://github.com/silicon-heaven/shvbroker-rs): provides `shvbroker`


## History aggregators

These are standalone server applications implementing SHV RPC History
functionality. They aggregate history logs.

No standalone implementation that fulfils the SVH 3.0 requirements is available,
yet (07.2024).


## CLI tools

These are generic command line tools that are designed to work with resources
exposed over SHV RPC. They can be used in scripts or just from interactive
session. They are tools for performing some specific operations.

### Method call

Generic way to call a single method. This is minimal functionality required to
get SHV RPC to work and thus it is provided by most of the implementations in
one way or the other.

* [shvapp](https://github.com/silicon-heaven/shvapp): provides `shvcall`
* [shvc](https://github.com/silicon-heaven/shvc): provides `shvc`
* [shvcall-rs](https://github.com/silicon-heaven/shvcall-rs): provides `shvcall`

### Signals retrieval

Tools to subscribe and receive SHV RPC signals.

* [shvc](https://github.com/silicon-heaven/shvc): provides `shvcsub`

### File copy

The tool to copy from and to SHV RPC File nodes.

* [shvc](https://github.com/silicon-heaven/shvc): provides `shvcp`

### Exchange interaction

Exchange nodes provide bi-directional stream of data. This is consistent with
CLI and thus it is beneficial to sometimes attach this stream to the console or
to some pipes.

* [shvc](https://github.com/silicon-heaven/shvc): provides `shvctio`

### ChainPack/CPON conversion

* [libshv](https://github.com/silicon-heaven/libshv): provides `cp2cp`
* [pyshv](https://github.com/silicon-heaven/pyshv): provides `pycpconv`


## Formal declaration of SHV tree

SHV tree provides API and thus it is beneficial to have a standard way to
describe this API in format that is readable by humans as well as computer. This
is provided by [SHVTree](https://github.com/silicon-heaven/shvtree) project.
