# Stream transport layers

Exchange of the data in streams is common abstraction that is provided by most
of the transport layers (such as TCP/IP). The important operation to support
streams in SHV RPC is to split stream to distinct messages. For this purpose
there are two different protocols defined; Block and Serial protocols.

## Block

The communication on bidirectional stream where delivery is ensured and checked.
The transport layer establishes and maintains point-to-point connection on its
own.

Message is transmitted in stream as one complete segment where message length is
sent before data. The receiving side first reads size and thus knows how much
data it should expect to receive. Message length is encoded as Chainpack
unsigned integer.

```
+--------+------+
| length | data |
+--------+------+
```

The transport error is detected if there is no byte received from other side for
more than 5 seconds during the message transfer.

Transport errors are handled by an immediate disconnect. It is expected that
connecting side is able to reestablish connection.

The primary transport layer used with this is TCP/IP, but this also applies to
Unix sockets or pair of pipes.


## Serial

This is communication over data stream with possible on the line errors (such as
data corruption).

The message data is encapsulated in start and stop byte and on link layers
without error checking it is also verified with CRC32. The dedicated bytes for
this are escaped in the message data.

```
+-----+------+-----------+---------+
| STX | data | ETX / ATX | *CRC32* |
+-----+------+-----------+---------+
```
* `STX` start of message `0xA2`
* `ETX` end of the message `0xA3`
* `ATX` abort the message `0xA4`
* `ESC` escape `0xAA`
  * `STX` in data will be coded as `ESC` `0x02`
  * `ETX` in data will be coded as `ESC` `0x03`
  * `ATX` in data will be coded as `ESC` `0x04`
  * `ESC` in data will be coded as `ESC` `0x0A`
* data - escaped message data
* CRC32 - escaped BigEndian CRC32 of `data` (same as used in IEEE 802.3 and
  [known as plain
  CRC-32](https://reveng.sourceforge.io/crc-catalogue/all.htm#crc.cat.crc-32-iso-hdlc))
  only on channels that do not provide data corruption prevention on its own and
  only after `ETX` (not after `ATX`).

The transport error is detected if there is no byte received from other side for
more that 5 seconds during the message transfer or when `STX` or `ATX` is
received before `EXT` or if `CRC32` do not match received data (on channels with
possible corruption such as RS232 and not TCP/IP).

Transport errors do not have to be handled explicitly because any subsequent
message can still be consistent. The invalid message should be just dropped.

The primary transport layer is RS232 with hardware flow control, but usage with
other streams, such as TCP/IP or Unix domain named socket, is also possible.

In some cases the restart of the connection might not be available. That is for
example when application just doesn't have rights for it or even when such
restart would not be propagated to the target, for what ever reason. To solve
this the empty message is reserved (that is message `STX ETX 0x0`). Once client
receives a valid empty message then it must drop any state it keeps for it.
