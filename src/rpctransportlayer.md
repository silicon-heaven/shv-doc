# RPC transport layer

## TCP / Stream
RPC communication is checked by TCP protocol. 
The socket is closed in the case of an error on the client or server side, and the whole communication is re-established. 
The error is also raised when no byte is received for 5 seconds on an incomplete message.

```
+--------+------+
| length | data |
+--------+------+
```

`data` consist of two parts
* [protocol_type](https://github.com/silicon-heaven/libshv/blob/353424a9b9b1943761a6a6ec50c1eb516a00877e/libshvchainpack/src/chainpack/rpc.h#L13) - 1 byte 
  * 01 - Chainpack
  * 02 - Cpon (deprecated)
  * 03 - Json (deprecated)
* `RpcData` - RpcMessage encoded according to protocol type, `ChainPack` in most cases.

## UDP
```
+------+
| data |
+------+
```

## RS232 / Serial
```
+-----+------+-----+-------+
| STX | data | ETX | CRC32 |
+-----+------+-----+-------+
```
* `STX` start of message `0xA2`
* `ETX` end of the message `0xA3`
* `ESTX` escaped STX code `0xA4`
* `EETX` escaped ETX code `0xA5`
* `ESC` escape `0xAA`
  * `STX` in data will be coded as `ESC` `ESTX`
  * `ETX` in data will be coded as `ESC` `EETX`
  * `ESC` in data will be coded as `ESC` `ESC`
* data - escaped message data (RPC message coded in chainpack)
* CRC32 - escaped BigEndian CRC32 of `data` ([POSIX CRC32](https://en.wikipedia.org/wiki/Cyclic_redundancy_check))

The message is considered invalid when no bytes are received for 5 seconds.

Special messages defined:
* `STX` `ETX` `CRC` - reset socket

