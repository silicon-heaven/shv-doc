# RpcMessage

`RpcMessage` is `IMap` with meta-data attached from [RpcValue](rpcvalue.md) point of view.

There are three types of messages used in the Silicon Heaven RPC communication.

## Request
Message used to call some method. Such messages is required to have `requestId`, `shvPath`, `method` name and optionally `parameters`. 
If the request goes via broker, the broker adds also `clientIds` attribute to the message meta-data to enable response back-propagation. 
Parameters can be any valid [RpcValue](rpcvalue.md).

## Response

Message sent by side that received some request. This message has to have
`requestId` and `result` or `error` but it can’t have method name. It has to copy `clientIds` from request message if they were present. 

## Signal

Spontaneous message sent without prior request and thus without `requestId`. It needs to specify `shvPath`, `method` and optionally `params` with any valid `RpcValue`.

`requestId` can be any unique number assigned by side that sends request initially. It is used to pair up requests with their responses. The common approach is to just use request message counter as `requestId`.

Mentioned `clientIds` are related to the broker and unless you are implementing a broker you need to know only that they are additional identifiers for the message alongside the `requestId` to pair requests with their responses and thus should be always included in the response message if they were present in the request.

The SHV path is used to select exact node of method in the SHV tree. The SHV tree is discussed in the next section.

```
+--------+----------+---------+
| Length | Coding   | Payload |
+--------+----------+---------+
```
* `Length` - UInt (in chainpack format without leading `PackingSchema` byte) - length of message excluding `Length`, len(Coding) + len(Payload)
* `Coding` - uint8_t 
  * `1` - ChainPack
* `Payload` - message payload, currently only `ChainPack` is supported.

## RPC
ChainPack RPC is relaxed form of JSON RPC, for example `jsonrpc` key is not required

Each RPC message is coded as `IMap` with `MetaTypeNameSpaceId == 1 (ChainPackNS)` and `MetaTypeId == 1 (ChainPackRpcMessage)`. Meta types are defined in `metatypes.h`

RequestId and RpcCallType are transmitted in message metadata. This allows us to get this information without parsing whole message. Broker can route messages according to this information without inspecting their content.

Defined ChainPack meta keys (`metatypes.h`):
```c++
struct Tag { enum Enum {Invalid = -1, MetaTypeId = 1, MetaTypeNameSpaceId, USER = 8 }; };
```
Defined ChainPackRpc message keys (`rpcmessage.h`):
```c++
struct Tag { enum Enum {RequestId = 8, Destination = 9, Method = 10};};
struct Key { enum Enum {Params = 1, Result, Error, ErrorCode, ErrorMessage, MAX};};
```
### examples:

RpcRequest Id=123, method="foo", params = {"a": 45, "b":  "bar", "c": [1,2,3]}
```
<8:123, 10:"foo">i{1:{"a":45,"b":"bar","c":[1,2,3]}}
10000001|00000001|10001111|00000011|00000001|10000110|01111011|00000010|10001011|00000011|01100110|01101111|01101111|00000011|10001110|00000011|00000001|01100001|01101101|00000001|01100010|10001011|00000011|01100010|01100001|01110010|00000001|01100011|10001101|00000011|01000001|01000010|01000011
```

RpcResponse SUCCESS result=42
```
<8:123>i{2:42u}
10000001|00000001|10001111|00000010|00000001|10000110|01111011|00000100|00101010
```
RpcResponse ERROR error_code=8, error_message="Method: 'foo' doesn't exist"
```
<8:123>i{3:i{1:8,2:"Method: 'foo' doesn't exist"}}
```

#### Examples
JSON `{"compact":true,"schema":0}` 27 bytes

MessagePack 18 bytes

ChainPack `|Map|String|7|'c'|'o'|'m'|'p'|'a'|'c'|'t'|True|String|6|'s'|'c'|'h'|'e'|'m'|'a'|TinyUInt(0)|TERM|` 21 bytes

## References and Links to Similar Projects

*  Concise Binary Object Representation (CBOR) [RFC8949](https://datatracker.ietf.org/doc/html/rfc8949)   

