# WebSockets

WebSockets are a natural fit for the RPC transport. There is no need to
establish any additional encoding or protocol layer. The RPC messages are
directly transmitted as WebSocket messages.

The [WebSocket's
subprotocol](https://websockets.spec.whatwg.org/#websocket-protocol) `shv3` is
used to identify SHV RPC.

## Pre-SHV3 WebSockets

Prior to SHV 3.0, WebSockets were used to transmit messages containing
arbitrarily fragmented bytes [stream with Block protocol](./stream.md#block).
This can still be supported if subprotocol `shv3` is *not* negotiated during
the WebSockets handshake.
