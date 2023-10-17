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


## Stream

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
more than 5 seconds during the message transfer.

Transport errors are handled by an immediate disconnect. It is expected that
connecting side is able to reestablish connection.

The primary transport layer used with this is TCP/IP but this also applies to
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
* CRC32 - escaped BigEndian CRC32 of `data` ([POSIX
  CRC32](https://en.wikipedia.org/wiki/Cyclic_redundancy_check)) only on
  channels that do not provide data corruption prevention on its own and only
  after `ETX` (not after `ATX`).

The transport error is detected if there is no byte received from other side for
more that 5 seconds during the message transfer or when `STX` or `ATX` is
received before `EXT` or if `CRC32` do not match received data (on channels with
possible corruption such as RS232 and not TCP/IP).

Transport errors do not have to be handled explicitly because any subsequent
message can still be consistent. The invalid message should be just dropped.

The primary transport layer is RS232 with hardware flow control, but usage with
other streams, such as TCP/IP or Unix domain named socket, is also possible.


## Datagram (DRAFT)

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

TODO: There are size limitations on the single message on OSes we might hit.
There is also an issue with reexecution of methods due to lost response (can't
identify if request or response got lost).


## CAN-FD (DRAFT)
`data` is split to N frames. There are 4 types of frame

Frame type | Value | Priority | Note
----       |----   |----      |----
SFM        | 0     | 0        | Single frame message, the highest priority
MFM_START  | 2     | 2        | First frame of multi frame message, higher priority than CONT
MFM_CONT   | 3     | 3        | Next frame(s) of multi frame message
MFM_END    | 1     | 1        | Last frame of multi frame message with CRC32

Priorities:
1. SFM has the highest priority, they should be used for notification
1. MFM_START should have higher priority than MFM_CONT to enable other client to start new session in parallel with existing session
1. MFM_CONT should have the lowest priority
1. MFM_END should have higher priority, than MFM_CONT to allow finishing session when other long message is sent.
1. MFM_END should have higher priority, than MFM_START to allow finishing session before new one is started.
1. Having these priorities we can transmit 2 frame long message (START + END) whilst other very long session is active.
1. When 2 long sessions run in parallel, then CONT messages from device with lower ID will be delivered first.

Scenarios:
1. SFM frame is lost
   * receiver doesn't get message
   * sender will timeout waiting for response 
1. MFM_START frame is lost
   * receiver will not get START message, so it cannot start session 
   * receiver will ignore all CONT and END messages, because there is no session SRC active
   * sender will timeout waiting for response 
1. MFM_CONT frame is lost
   * receiver will get END message with invalid CRC
   * receiver will cancel active session
   * sender will timeout waiting for response 
1. MFM_END frame is lost
   * receiver will not get END message
   * receiver will timeout and cancel active session
   * sender will timeout waiting for response 
1. MFM_END frame has invalid CRC
   * receiver will get END message with invalid CRC
   * receiver will cancel active session
   * sender will timeout waiting for response 
1. More senders will send MFM simultaneously to one receiver
   * will receive first START message and starts receive session for sender with SRC address
   * receiver will ignore all messages with senders with different SRC than one from current session
   * first sender will send message
   * other senders will timeout waiting for response 
1. Receiver has active session and MFM_START message is received
   * receiver can parse first frame and decode SHV request ID. Then it can create RPC response `device bussy` and reply with RPC Exception. This can avoid waiting till timeout on the sender side. Sender also will know that receiver is connected and alive, what is more information than the timeout can provide.

### CAN ID structure
```
+---------------+----------------+---------------------------+
| 1 bit channel | 2 bit priority | 8 bit src device address  |
+---------------+----------------+---------------------------+
```
* priority
* src device address - address of sender device.

**Frames**
* `SFM` - Single frame message. SFM does not need CRC, since it is part of CAN frame already
* `MFM` - Multi frame message, consisting of `MFM_START` + `MFM_CONT`* + `MFM_END` 

#### SFM
```
+---------------------------+------------------------+----------------------+ 
| 8 bit dest device address | 8 bit frame type (SFM) | 8 bit payload length |
+---------------------------+------------------------+----------------------+ 
+----------------------+
| 0 - 61 bytes payload |
+----------------------+
```

#### MFM_START
```
+---------------------------+------------------------------+
| 8 bit dest device address | 8 bit frame type (MFM_START) |
+---------------------------+------------------------------+
+------------------+
| 62 bytes payload |
+------------------+
```
Note that `MFM_START` can have only 62 bytes of payload, 61 bytes can fit to single `SFM`.

#### MFM_CONT
```
+---------------------------+-----------------------------+----------------------+----------------------+ 
| 8 bit dest device address | 8 bit frame type (MFM_CONT) | 8 bit payload length | 8 bit frame counter  |
+---------------------------+-----------------------------+----------------------+----------------------+ 
+-----------------------+
| 58 - 60 bytes payload |
+-----------------------+
```
Note that `MFM_CONT` cannot have less than 58 bytes of payload, 57 bytes can fit to `MFM_END`.


#### MFM_END
```
+---------------------------+----------------------------+----------------------+ 
| 8 bit dest device address | 8 bit frame type (MFM_END) | 8 bit payload length |
+---------------------------+----------------------------+----------------------+ 
+----------------------+
| 0 - 57 bytes payload |
+----------------------+
+---------------+
| 32 bit CRC LE |
+---------------+
```
CRC32 must not be split between frames.


TODO: We need some packet that signals device overload (no ability to process new messages for some time)


