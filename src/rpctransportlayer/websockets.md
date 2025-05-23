# WebSockets

WebSockets pretty much ideally fit the RPC transport requirements and thus there
is no need to establish any additional encoding or protocol layer. The RPC
messages can be directly transmitted as WebSocket messages.

The WebSocket's subprotocol `shv3` is used to identify SHV RPC.

## Pre-SHV3 WebSockets

Before SHV 3.0 WebSockets were used to transmit messages containing arbitrary
fragmented bytes from [stream with Block protocol](./stream.md#block). This can
still be supported if subprotocol `shv3` is not selected during WebSockets
handshake.
