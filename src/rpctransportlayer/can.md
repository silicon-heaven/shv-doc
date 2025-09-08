## CAN-FD transport layer

| ❗ This document is in DRAFT stage |
|------------------------------------|

The communication over CAN-FD (Control Area Network) bus. Compared to other
transport layers, that are point-to-point, this is bus and thus it must not only
provide message transfer but also a connection tracking and multiplexing.

CAN-FD has the following abilities:
* Delivery of the CAN frames is not ensured (it can be lost)
* Single CAN frame can be transmitted multiple times
* CAN frames are delivered in order based on CAN ID priority and never out of
  order
* Collision is resolved by avoiding it based on the CAN ID (lower has higher
  priority)
* CAN frames correctness is ensured with CRC32
* Flow control is handled by CAN overload frame

CAN(-FD) supports different types of the frames:
* Data frames: regular frames used to transfer data in blocks of 0, 1, 2, 3, 4,
  5, 6, 7, 8, 12, 16, 20, 24, 32, 48, 64 bytes.
* Remote request frames: frame that caries no data and data length (`0x0` -
  `0xf`) is just informative.

CAN bus is designed for control applications but SHV RPC is rather configuration
and management interface and thus the design here prioritizes fair queueing over
message delivery deadline guaranties. This is because we expect that SHV RPC
will overcommit the bus (contrary to common control applications).

CAN bus has limited bandwidth and thus it is not desirable in most cases to emit
all signals device has (contrary to standard SHV RPC device design). The SHV
native way to introduce filtering is to use RPC Broker and thus it is highly
suggested that devices on CAN bus should expose RPC Broker to it.

### CAN ID

CAN implements collision avoidance scheme where all devices want to transmit
their ID (and other bits) during arbitration phase and the one with lowest ID
wins the time slot so it can send its data (not ultimately true if extended ID
is used, but SHV uses 11 bits only).

CAN ID can be either 29 bits or 11 bits. SHV RPC uses exclusively only 11 bits
ID.

```
+------------+------------+-----------+------------------+
| SHVCAN [1] | Unused [1] | First [1] | Peer address [8] |
+------------+------------+-----------+------------------+
```

* `SHVCAN` is always `1` for SHV. This allows bus to be shared with some other
  traffic that is not SHV defined.
* `Unused` is always `1` at the moment. It is reserved for the future expansion.
* `First` should be set to `1` for the first frame of the message and `0`
  otherwise.
* `Peer address` is an unique address of the peer. Thus every peer on the bus
  must have an unique address if it wants to participate in SHV communication.

_Note: The advantage of using only 11 bits is with CAN-FD where initial
arbitration runs in slower speed and by not using 29 bits it will be shorter and
thus communication faster._

### Connection

Every frame has address of the peer sending the frame in its CAN ID. For the
destination peer the first byte of data is used. Thus data frame of zero length
is invalid in this protocol.

The connection is established by first message. It is required to use
[**ResetSession**](../rpctransportlayer.md) message to ensure that even if
previous connection wasn't terminated that the state is reset (the implicit
connection creation doesn't provide full connection tracking functionality).

The connection is terminated by special last frame that contains only
destination peer (thus frame of data size 1).

### Fragmentation

SHV RPC Message has to be fragmented to multiple frames if it is longer than one
frame can carry. This means that multiple frames have to be associated in some
way to detect if unbroken sequence was received. This is ensured by counter in
second data byte. With first byte allocated for the destination and second byte
for the counter the fragment can contain from 1 to 62 bytes of message data.

The second data byte also serves as last frame signalization. The most
significant bit in this byte is set for last frame and unset otherwise. Thus
only seven bits are actually used as counter. The last frame is the frame
containing last bytes of the message but due to limited CAN-FD frame sizing the
frame might be stuffed by `0x00` at the end, these bytes should be removed by
SHV CAN-FD implementation (that is why last byte of the SHV RPC Message can't be
null byte), but only if the complete message is longer than 8 bytes.

The first frame is identified by `First` bit in the CAN ID. The counter can be
of any value (between `0x00` and `0x7f`) for this frame. The only requirement is
that it is not the same as for the previously used first frame (see the next
chapter about flow control for the reasoning). The counter must be increased by
one for every subsequent frame where value `0x7f` wraps to `0x00`. The message
receiving peer must verify sequence of the counter to verify if all frames were
received. The retrieval of message is aborted if counter sequence is broken (be
aware that receiving the same message multiple times in the row can occur and is
valid). Such message is just lost and no error should be reported for that. This
behavior can be also used for message abort (intentional sequence breaking).

### Flow Control

The throughput of CAN-FD is typically low enough that even low-end devices
should be able to process all incoming data without issue. However, problems may
arise when multiple connections are forwarded to a backend that doesn't support
multiplexing (such as [stream transport](./stream.md)), which is often the case
with RPC brokers. To address this, SHV CAN-FD includes a simple flow control
mechanism.

A peer is allowed to send the first data frame of a message, but before it can
continue sending the rest of the message, and the first data frame of the next
message, it must receive an acknowledgment. This acknowledgment is a message
containing only the destination byte and a copy of the second byte from the
original first data frame. Notably, only the first frame of each message is
acknowledged; subsequent data frames as well as next first data frame are sent
without further confirmation.

Peer can send first frame without any previous acknowledgment after startup.
After that it has to wait for acknowledgment to give other peer time to handle
the other messages. The message can be aborted before acknowledgment by sending
a different first frame (this is also the same pattern that appears on device
reset).

The peer is allowed to resend first frame if no acknowledgment is received in
reasonable amount of time to cover case when destination peer restarted or frame
got lost.

### Address Discovery

It is beneficial to be able to discover peers on the CAN Bus. This is done by
remote request frames. For discovery we differentiate between two different
peers, those accepting new connections and those that do not.

The discovery is triggered by remote request frame with data length `0x5`,
`0x6`, or `0x7`. Peers that are accepting new connections must respond with
remote request frame with data length `0x1` on discovery requests `0x5` and
`0x7`. Peers that are **not** accepting new connections must respond with remote
request frame with data length `0x2` on discovery requests `0x6` and `0x7`.

The CAN ID used for the remote request frames are not considered as first frames
(to prioritize this type of traffic). The address used is always the peer's own
address.

For real time discovery it is desirable that any new peer that starts accepting
connections sends remote request frame with data length `0x1` without any
previous discovery request.

### Dynamic Address

The SHV CAN-FD relies on every peer on the bus to have an unique address but
there is only 256 of them and when unknown network is being accessed (or even
probed) some address has to be used without knowledge of the free ones. For that
purpose a dynamic address acquisition is provided. Addresses from `0x00` to
`0x7f` can be assigned statically but addresses from `0x80` to `0xff` have to be
always dynamically acquired. This ensures that dynamically acquired address is
not statically assigned to the peer that is currently offline.

The dynamic acquisition happens by random selection. The peer will select
randomly address in the dynamic range and sends eight subsequent remote request
frames with data length `0x0` and CAN ID with First set to `0`. These remote
request frames have to be transmitted on the CAN bus without receiving either
concurrent address acquisition or address announce with same source.

Peers with address acquired with dynamic acquisition have to listen for remote
request frames with data length `0x0` and send their respective address announce
remote request frame to prevent address collision.

### Reference

Be aware that only addresses between `0x00` and `0x7f` can be assigned
statically!

The frame exchange sending a message:

```
sender                  receiver            sender                     receiver
------                  --------            ------                     --------
   |                        |                 |                           |
   | First data frame       |                 | First and last data frame |
   |----------------------->|                 |-------------------------->|
   |                        |                 |                           |
   | Acknowledgment frame   |                 | Acknowledgment frame      |
   |<-----------------------|                 |<--------------------------|
   |                        |                             ...
   | Subsequent data frames |                 | First data frame          |
   |----------------------->|                 |-------------------------->|
   |----------------------->|                             ...
   |                        |
   | Last data frame        |           
   |----------------------->|           
            ...
   | First data frame       |           
   |------------------ ---->|           
            ...
```

The first data frame has format (where `Source` and `Destination` are addresses
of the peers communication):

```
CAN ID (bits)            Frame data (bytes)
+---+---+---+--------+   +-------------+---------+------------
| 1 | 1 | 1 | Source |   | Destination | Counter | Data ...
+---+---+---+--------+   +-------------+---------+------------
```

The acknowledgment frame has format (where `Source` and `Destination` have the
same meaning as for the first data frame and `Counter` is its copy):

```
CAN ID (bits)                 Frame data (bytes)
+---+---+---+-------------+   +--------+---------+
| 1 | 1 | 0 | Destination |   | Source | Counter |
+---+---+---+-------------+   +--------+---------+
```

The subsequent data frames have format (where `Source` and `Destination` have
the same meaning as for the first data frame and `Counter` increases by one for
every subsequent frame):

```
CAN ID (bits)            Frame data (bytes)
+---+---+---+--------+   +-------------+---------+------------
| 1 | 1 | 0 | Source |   | Destination | Counter | Data ...
+---+---+---+--------+   +-------------+---------+------------
```

The last data frame has format (where `Source` and `Destination` have the same
meaning as for the first data frame and `Counter` increased by one relative to
the last subsequent frame or the first frame if there were no subsequent
frames):

```
CAN ID (bits)            Frame data (bytes)
+---+---+---+--------+   +-------------+----------------+------------
| 1 | 1 | 0 | Source |   | Destination | 0x80 + Counter | Data ...
+---+---+---+--------+   +-------------+----------------+------------
```

The first frame that is also at the same time the first frame (the message fits
to the 62 bytes):

```
CAN ID (bits)            Frame data (bytes)
+---+---+---+--------+   +-------------+----------------+------------
| 1 | 1 | 1 | Source |   | Destination | 0x80 + Counter | Data ...
+---+---+---+--------+   +-------------+----------------+------------
```

The connection terminate frame (the `Source` and `Destination` can be flipped if
termination/disconnect is performed by other side):

```
CAN ID (bits)            Frame data (bytes)
+---+---+---+--------+   +-------------+
| 1 | 1 | 1 | Source |   | Destination |
+---+---+---+--------+   +-------------+
```

Remote request frames:

```
CAN ID (bits)
+---+---+----------+---------+
| 1 | 1 | Priority | Address |
+---+---+----------+---------+
```

| Data length field | Priority bit | Usage                                                     |
|-------------------|--------------|-----------------------------------------------------------|
| 0b0000            | 1            | Address acquisition                                       |
| 0b0001            | 0            | Address announce for peers accepting connections          |
| 0b0010            | 0            | Address announce for peers not accepting new connections  |
| 0b0101            | 0            | Address discovery for peers accepting connections         |
| 0b0110            | 0            | Address discovery for peers not accepting new connections |
| 0b0111            | 0            | Address discovery for all peers                           |
