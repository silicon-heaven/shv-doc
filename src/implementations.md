# Implementations

This is the list of projects implementing at least ChainPack and/or CPON data
formats. Most of the provide the abstraction on SHV RPC.

| Project                                                                            | Language   | Note                                               |
|------------------------------------------------------------------------------------|------------|----------------------------------------------------|
| [libshv C++](https://github.com/silicon-heaven/libshv)                             | C++        | Reference implementation                           |
| [libshv C](https://github.com/silicon-heaven/libshv/tree/master/libshvchainpack/c) | C          | malloc() free implementation of CPON and ChainPack |
| [SHVC](https://gitlab.com/elektroline-predator/shvc)                               | C          | Complete implementation of SHV RPC                 |
| [libshv-js](https://github.com/silicon-heaven/libshv-js)                           | JavaScript |                                                    |
| [pySHV](https://gitlab.com/elektroline-predator/pyshv)                             | Python     |                                                    |
| [chainpack-rs](https://github.com/silicon-heaven/chainpack-rs)                     | Rust       |                                                    |
| [libshv D](https://github.com/silicon-heaven/libshv/tree/master/libshvchainpack/d) | D          | not maintained currently                           |

These are additional utility projects that are handy to know about when working
with SHV:

| Project                                              | Description                                                    |
|------------------------------------------------------|----------------------------------------------------------------|
| [shvspy](https://github.com/silicon-heaven/shvspy)   | The Qt GUI for listing and accessing the SHV RPC network       |
| [shvcli](https://github.com/silicon-heaven/shvcli)   | CLI interface for listing and accessing SHV RPC network        |
| [shvtree](https://github.com/silicon-heaven/shvtree) | Formal declaration of SHV RPC node tree and data format scheme |
