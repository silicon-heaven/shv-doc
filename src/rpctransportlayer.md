# RPC transport layer

RPC messages can be transmitted over different layers. Any message transport
layer that guarantees the following functionalities can be used for SHV RPC:

* Messages need to be transferred completely without error or transport error
  needs to be detected.
* Any message can be lost without transport error reporting error.
* Communication needs to be point-to-point, or transport layer needs to emulate
  that.
* Messages can be of any size

Exchanged messages have `data` always start with a single byte identifying the
format of the subsequent bytes in the message (if any):

* `00` - ResetSession
* `01` - Chainpack encoded [RPC Message](./rpcmessage.md)
* `02` - Cpon encoded [RpcMessage](./rpcmessage.md) (deprecated)
* `03` - Json encoded [RpcMessage](./rpcmessage.md) (deprecated)

`ResetSession` message is used to reset current session. Basically it works in
the same manner as socket reconnection, but it can be used also on layers
without connection tracking like serial port. When `ResetSession` is received,
then receiver should clear the peer's state. `ResetSession` message can be used
by client to log-out or by broker to kick-out client. `ResetSession` message
contains only Format part, the message is excluded. It is `01 00` on stream
layer or `STX 00 ETX` or `STX 00 ETX D2 02 EF 8D` (if CRC is on) on serial
layer. This message must be sent before `hello` on layers without connection
tracking (TTY, CAN). It might be optionally sent also on other layers like
socket.

There are protocols defined for some common transport layers that do not support
message transport layer as required by SHV RPC. This includes the following:

- [Stream transport layers](./rpctransportlayer/stream.md)
- [CAN Bus](./rpctransportlayer/can.md)
