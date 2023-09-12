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
* `ESTX` escaped STX code `0xA4`
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

## CAN-FD (DRAFT)
`data` is split to N frames. There are 4 types of frame

Scenarios:
1. SFM frame is lost
   * receiver doesn't get message
   * sender will timeout waiting for response 
1. MFM_START frame is lost
   * receiver will not get START message, so it cannot start session 
   * receiver will ignore all CONT and END messages, because there is not session SRC active
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
   * othert senders will timeout waiting for response 
1. Receiver has active session and MFM_START message is received
   * receiver can parse first frame and decode SHV request ID. Then it can create RPC response `device bussy` and reply with RPC Exception. This can avoid waiting till timeout on the sender side. Sender also will know that receiver is connected and alive, what is more information than the tomeout can provide.

Priorities:
1. SFM has highest priority, they should be used for notification
1. MFM_START should have higher priority than MFM_CONT to enable other client to start new session in paralel with existing session
1. MFM_CONT should have the lowest priority
1. MFM_END should have higher priority, than MFM_CONT to allow finishing session when other long message is sent.
1. MFM_END should have higher priority, than MFM_START to allow finishing session before new one is started.
1. Having this priorities we can transmit 2 frame long message (START + END) whilst other very long session is active.
1. When 2 long sessions run in paralel, then CONT messages from device with lower ID wil be delivered first.

Frame type | Value | Priority | Note
----|----|----|----
SFM | 0 | 0 | Single frame message, highest priority
MFM_START | 2 | 2 | First frame of multi frame message, higher priority than CONT
MFM_CONT | 3 | 3 | Next frame(s) of multi frame message
MFM_END | 1 | 1 | Last frame of multi frame message with CRC32

### CAN ID structure
```
+---------------+----------------+---------------------------+
| 1 bit channel | 2 bit priority | 8 bit src device address  |
+---------------+----------------+---------------------------+
```
* priority
* src device address - address of sender device.

### Frames
* frame type
  * SFM - Single frame message. SFM does not need CRC, since it is part of CAN frame already
  * MFM - Multi frame message, consisting of `MFM_START` + `MFM_CONT`* + `MFM_END` 

```
SFM
+---------------------------+------------------------+----------------------+ 
| 8 bit dest device address | 8 bit frame type (SFM) | 8 bit payload length |
+---------------------------+------------------------+----------------------+ 
+----------------------+
| 0 - 61 bytes payload |
+----------------------+

MFM_START
+---------------------------+------------------------------+
| 8 bit dest device address | 8 bit frame type (MFM_START) |
+---------------------------+------------------------------+
+------------------+
| 62 bytes payload |
+------------------+

MFM_CONT
+---------------------------+-----------------------------+----------------------+ 
| 8 bit dest device address | 8 bit frame type (MFM_CONT) | 8 bit payload length |
+---------------------------+-----------------------------+----------------------+ 
+-----------------------+
| 58 - 61 bytes payload |
+-----------------------+

MFM_END
+---------------------------+----------------------------+----------------------+ 
| 8 bit dest device address | 8 bit frame type (MFM_END) | 8 bit payload length |
+---------------------------+----------------------------+----------------------+ 
+----------------------+
| 4 - 61 bytes payload |
+----------------------+
+------------+
| 32 bit CRC |
+------------+
```

CRC32 must not be split between frames.

TODO: MFM_CONT needs some counter to prevent frames duplicates due to packet duplication.
TODO: We need some packet that signals device overload (no ability to press new messages for some time)


