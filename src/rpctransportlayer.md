# RPC transport layer

RPC messages can be transmitted over different layers. What they have in common
is that they transfer messages in the following format:

```
+--------+-------------+
| Format | RPC Message |
+--------+-------------+
```

[Format is a single
byte](https://github.com/silicon-heaven/libshv/blob/353424a9b9b1943761a6a6ec50c1eb516a00877e/libshvchainpack/src/chainpack/rpc.h#L13)
that signals the encoding of the following data:
* `01` - Chainpack
* `02` - Cpon (deprecated)
* `03` - Json (deprecated)

RPC Message is actual [transmitted message](rpcmessage.md).

Transport layers needs to ensure the following guaranties:
* Messages need to be transferred completely without error or transport error
  needs to be detected.
* Any message can be lost without transport error reporting error.
* Communication needs to be point-to-point or transport layer needs to emulate
  that.


## TCP / Stream

The communication on bidirectional stream where delivery is ensured and checked.
The transport layer establishes and maintains point-to-point connection on its
own.

Message is transmitted in stream as one complete segment where message length is
sent before data. The receiving side first reads size and thus knows how much
data it should expect to receive. Message length is encoded as Chainpack
unsigned integer.

```
+--------+--------------+
| length | message data |
+--------+--------------+
```

The transport error is detected if there is no byte received from other side for
more than 5 seconds.

Transport errors are handled by an immediate disconnect. It is expected that
connecting side is able to reestablish connection.

The primary transport layer used with this is TCP/IP but this also applies to
Unix sockets or pair of pipes.


## UDP / Datagram

The datagram communication where delivery and order is not ensured. Datagrams
have known size and ensure data consistency.

```
+--------------+
| message data |
+--------------+
```

The transport error is detected if there is no complete message received for
more than 5 seconds.

Transport errors are handled by immediate disconnect, but it is possible that
this disconnect is only one sided because most of the datagram transport layers
do not establish connection.


## RS232 / Serial

This is communication over data stream with possible on the line errors (such as
data corruption). It is also expected that application can't perform reconnect
of this data stream.

The message data is encapsulated in start and stop byte and verified with CRC32.
The dedicated bytes for this are escaped in the message data.

```
+-----+------+-----+-------+
| STX | data | ETX | CRC32 |
+-----+------+-----+-------+
```
* `STX` start of message `0xA2`
* `ETX` end of the message `0xA3`
* `ESTX` escapd STX code `0xA4`
* `EETX` escaped ETX code `0xA5`
* `ESC` escape `0xAA`
  * `STX` in data will be coded as `ESC` `ESTX`
  * `ETX` in data will be coded as `ESC` `EETX`
  * `ESC` in data will be coded as `ESC` `ESC`
* data - escaped message data
* CRC32 - escaped BigEndian CRC32 of `data` ([POSIX
  CRC32](https://en.wikipedia.org/wiki/Cyclic_redundancy_check))

The transport error is detected if there is no byte received from other side for
more than 5 seconds or when `STX` is received before `EXT` or if `CRC32` does
not match.

Transport errors are handled by special empty message (`STX ETC CRC`). This
message when received should cause reset of receiver side state machine and thus
termination of any undelivered message.
