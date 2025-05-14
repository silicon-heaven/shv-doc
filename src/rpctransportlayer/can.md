## CAN-FD transport layer

| ❗ This document is in DRAFT stage |
|------------------------------------|

The communication over CAN (Control Area Network) bus. Compared to other
transport layers that are point-to-point this is bus and thus it must not only
provide message transfer but also a connection tracking.

CAN has the following abilities:
* Delivery of the CAN frames is not ensured (it can be lost)
* Single CAN frame can be transmitted multiple times
* CAN frames are delivered in order based on CAN ID priority and never out of
  order
* Collision is resolved by avoiding it based on the CAN ID (lower has higher
  priority)
* CAN frames correctness is ensured with CRC32
* Flow control is handled by CAN overload frame

CAN supports few different types of the frames:
* Data frames: regular frames used to transfer data
* Remote frames: frame that caries no data and data length (`0x0` - `0xf`) is
  just informative. SHV RPC uses these as following signal frames:
  * Message abort frame with data length `0x0`
  * Device advertisement frame with data length `0x1`
  * Device discover frame with data length `0x2`

CAN bus is designed for control applications but SHV RPC is rather configuration
and management interface and thus the design here prioritizes fair queueing over
message delivery deadline guaranties. This is because we expect that SHV RPC
will overcommit the bus (contrary to common control applications).

CAN bus has limited bandwidth and thus it is not desirable in most cases to emit
all signals device has (contrary to standard SHV RPC device design). The SHV
native way to introduce filtering is to use RPC Broker and thus it is highly
suggested that devices on CAN bus should expose RPC Broker to it.
##
### CAN ID

CAN ID can be either 29 bits or 11 bits. SHV RPC uses exclusively only 11 bits
ID.

```
+-------------+-----------+---------+--------------------+
| NotLast [1] | First [1] | QoS [1] | Device address [8] |
+-------------+-----------+---------+--------------------+
```

* `NotLast` is `0` when no subsequent CAN frame will follow this one and `1`
  otherwise. This prioritizes message termination on the bus.
* `First` is `1` when this is initial CAN frame and `0` otherwise. This penalizes
  start of the new message and thus prefers SHV RPC message finish.
* `QoS` should be flipped on every SHV RPC message sent. It ensures that devices
  with high CAN ID (low priority) get chance to transmit their messages. It is
  allowed that device that is quiet for considerable amount of time to set it to
  any state (commonly beneficial for that device).
* `Device address` this must be unique address of the device transmitting CAN
  frame or bitwise not of it when `QoS` is `1`. The addresses `0x0` and `0xff`
  are reserved. The bit flipping of the device address based on the `QoS`
  ensures that high priority CAN IDs in QoS `1` are low priority ones in the QoS
  `0` and vice versa, thus devices should get somewhat equal chance to transmit
  their messages.

_Note: The advantage of using only 11 bits is with CAN-FD where initial
arbitration runs in slower speed and by not using 29 bits it will be shorter and
thus communication faster._

### First data byte

The first data byte in the CAN frame has special meaning.

For CAN frames with `First` bit set (`1`) in CAN ID the first byte must be
destination device address. This identifies the device this SHV RPC message is
intended for. 

For CAN frames with `First` bit unset (`0`) in CAN ID the first byte must be
sequence number that starts with `0x0` for the second CAN frame (third is `0x1`
and so on). If sequence number reaches `0xff` then it just simply wraps around
to `0x0`.

The complete and valid SHV RPC message thus starts with CAN frame with `First`
set, continues with CAN frames where `First` is not set and `NotLast` is set and
first data byte is sequence, and terminates by CAN frame where `NotLast` is not
set. The consistency of the message (that no CAN frame is lost) is ensured with
counter in first data byte.

Transport error is detected if CAN frame with `First` not set in CAN ID is
received with byte having number in first data byte out of order (while ignoring
frames with same number as the last received CAN frame from that specific
device). Such SHV RPC message is silently dropped and error is ignored as
subsequent messages can be consistent again without any action.

The message can be terminated by CAN RTR frame (Remote transmission request)
with both `NotLast` and `First` unset and data length set to `0x0`.

### Connection

Connection between devices is automatically established with first message being
exchanged. There can be only one connection channel between two devices. To
disengage it the `ResetSession` message can be sent (that is regular CAN frame
with `NotLast` not set and `First` set in CAN ID and data containing only byte
with destination device address and one `00` byte).

### Broadcast

Due to the CAN bus bandwidth limitations it is suggested to expose SHV RPC
Broker instead of just plain device, but sometimes it is beneficial to not add
the signal filtering and instead to automatically broadcast signals. Such
signals are not intended for any specific device and are just submitted on the
bus with special destination device address `0xff`.

The only allowed SHV RPC message type for destination device address `0xff` is
[RpcSignal](../rpcmessage.md#rpcsignal). Other message types received with this
destination device address must be ignored and devices should not send them.

Handling of the broadcast signals is up to the application it receives it.
SHV RPC Brokers will propagate them further if given device is mounted and
subscribed.

One concrete example when this is beneficial is for date and time
synchronization device. Such device can send signals with precise time to let
other devices to synchronize with it.

### SHV Device discovery

SHV devices on CAN bus must be possible to discover to not only be able to
dynamically mount them to SHV RPC Broker but also to actively diagnose the CAN
bus.

Once device is ready to receive messages on CAN bus it should send CAN RTR frame
(Remote transmission request) with data length set to `0x1` (the CAN ID should
have `NotLast` not set and `First` set which applies to all CAN RTR frames).
This ensures that it gets discovered as soon as possible.

Device that wants to perform discovery can send CAN RTR frame (Remote
transmission request) with data length set to `0x2`. Device that receives this
CAN frame must respond with CAN RTR frame with data length set to `0x1`.

### Address collision resolution

The standard deployment should prevent address collisions by allocating them for
the whole bus before deployment, but there is also a use case for on site ad hoc
connection and in such situation it is unclear what address should be chosen.
The device should select random unallocated address but that won't prevent
collision, only minimize them.

All CAN devices (that includes SHV clients) must listen for others using their
address and must send CAN RTR (Remote transmission request) frame with size set
to `0x3` when they receive CAN frame they did not send. CAN devices with same
dynamic address receiving this RTR frame must choose a different one.

The best practice is to choose address from upper range (close to 255) and
initially send dummy CAN RTR frame with size set to `0xf`. Do this in 200ms
intervals five times. The communication with other CAN devices can start if no
CAN RTR frame with size `0x3` is received in the meantime.

### CAN RTR frames index

This is full list of all different CAN RTR frames used in the protocol:

| Data length field | Usage                        |
|-------------------|------------------------------|
| 0x0               | SHV message termination      |
| 0x1               | Device presence announcement |
| 0x2               | Device discovery request     |
| 0x3               | Address collision notice     |
| 0xf               | Dummy frame with no effect   |
