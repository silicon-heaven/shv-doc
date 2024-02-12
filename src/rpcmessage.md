# RpcMessage

`RpcMessage` is `IMap` with meta-data attached from [RpcValue](rpcvalue.md) point of view.

There are three kinds of RPC messages defined:
* [RpcRequest](#rpcrequest)
* [RpcResponse](#rpcresponse)
* [RpcSignal](#rpcsignal)

RPC message can have meta-data attribute defined.

Attribute number | Attribute name      | Description
----------------:|---------------------|------------
1                | MetaTypeId          | Always equal to `1` in case of RPC message 
2                | MetaTypeNameSpaceId | Always equal to `0` in case of RPC message, may be omitted.
8                | RequestId           | Every RPC request must have unique number per client. Matching RPC response will have the same number. 
9                | ShvPath             | Path on which method will be called. 
10               | Method              | Name of called RPC method 
11               | CallerIds           | Internal attribute filled by broker in request message to distinguish requests with the same request ID, but issued by different clients.  
13               | RespCallerIds       | Internal attribute filled by broker in response message to enable support for multi-part messages and tunneling. <https://github.com/silicon-heaven/libshv/wiki/multipart-messages>
14               | Access              | Access granted by broker to called `shvPath` and `method` to current user. 
16               | UserId              | ID of user calling RPC method filled in by broker.
17               | AccessLevel         | Reserved, integer value, it will be used in next API version for chained brokers access capping  
18               | Part                | Reserved, it will be used in next API version for multi-part messages   <https://github.com/silicon-heaven/libshv/wiki/multipart-messages>

Second part of RPC message is `IMap` with following possible keys.


Key | Key name | Description
---:|----------|------------
1   | Params   | Optional method parameters, any [RpcValue](rpcvalue.md) is allowed.
2   | Result   | Successful method call result, any [RpcValue](rpcvalue.md) is allowed.
3   | Error    | Method call exception, see [RPC error](#rpc-error) for more details

`RequestId` can be any unique number assigned by side that sends request
initially. It is used to pair up requests with their responses. The common
approach is to just use request message counter as request ID.

`CallerIds` are related to the broker and unless you are implementing a
broker you need to know only that they are additional identifiers for the
message alongside the `RequestId` to pair requests with their responses and thus
should be always included in the response message if they were present in the
request.

The `ShvPath` is used to select exact node of method in the SHV tree. 

`Access` is part of access control. It is assigned to request by broker according to user rights.
Multiple grants can be specified and separated by comma. 

## RpcRequest

Message used to invoke remote method.

Methods invoked using this request needs to be idempotent, because RPC
transport layers do not ensure deliverability. You might also want to try to
send request again when you receive no response because of this.

Attributes

Attribute | Required | Note
----------|----------|-----
`MetaTypeId`   | yes | Always set to `1`
`RequestId`    | yes |
`ShvPath`      | yes |
`Method`       | yes |
`RevCallerIds` | no  | If tunneling or multi-part message is needed
`CallerIds`    |     | Attribute managed by broker
`Access`       |     | Attribute managed by broker
`UserId`       |     | Attribute managed by broker

Keys

Key      | Required | Note
---------|----------|-----
`Params` | no       | Any valid [RpcValue](rpcvalue.md)

**Examples**

RPC call invocation, method `switchLeft` on 
path `test/pme/849V` with request ID `56` and parameter `true`. 
```
<1:1,8:56,9:"test/pme/849V",10:"switchLeft">i{1:true}
```

## RpcResponse

Response to [RpcRequest](rpcrequest.md)

Attributes

Attribute | Required | Note
----------|----------|-----
`MetaTypeId`   | yes | Always set to `1`
`RequestId`    | yes | ID of matching `RpcRequest`
`RevCallerIds` | no  | Must be copied from `RpcRequest` if present.
`CallerIds`    | no  | Must be copied from `RpcRequest` if present.

Keys

Key      | Required | Note
---------|----------|-----
`Result` | yes      | Required in case of successful method call result, any [RpcValue](rpcvalue.md) is allowed.
`Error`  | yes      | Required in case of method call exception, see [RPC error](#rpc-error) for more details.

### RPC Error

RPC Error is `IMap` with following keys defined

Key | Key name  | Required | Description
---:|---------- |----------|-------
1   | `Code`    | yes      | Error code
2   | `Message` | no       | Error message string
3   | `Data`    | no       | Arbitrary payload, can be used for example for exception localization aditional info.
specific and it is not defined by SHV RPC.

Error codes

Value | Name  | Description
-----:|-------|----------
0  | `NoError`             | 
1  | `InvalidRequest`      | The `RpcValue` sent is not a valid RPC Request object.
2  | `MethodNotFound`      | The method does not exist / is not available.
3  | `InvalidParams`       | Invalid method parameter(s).
4  | `InternalError`       | Internal RPC error.
5  | `ParseError`          | Invalid `ChainPack` was received.
6  | `MethodCallTimeout`   | 
7  | `MethodCallCancelled` | 
8  | `MethodCallException` | 
9  | `Unknown`             | 
10 | `LoginRequired`       | Method call without previous successful login.
32 | `UserCode`            | 

**Examples**

Successful response to method call from example above
```
<1:1,8:56>i{2:true}
```
Exception when unknown method is called
```
<1:1,8:11>i{3:i{1:8,2:"method: foo path:  what: Method: 'foo' on path 'shv/cze' doesn't exist"}}
```

## RpcSignal

Spontaneous message sent without prior request and thus without `RequestId`. 
It is used mainly notify clients that some technological value had changed without need to poll.

Attributes

Attribute | Required | Note
----------|----------|-----
`MetaTypeId`   | yes | Always set to `1`
`ShvPath`      | yes | Property path
`Method`       | yes | Signal name, the property changes have obviously method `chng`
`Access`       |     | Minimal access level needed for this method. The `"rd"` is used if not specified.

Keys

Key      | Required | Note
---------|----------|-----
`Params` | no       | Any valid [RpcValue](rpcvalue.md)

**Examples**

Spontaneous notification about change of `status/motorMoving` property to `true`.
```
<1:1,9:"shv/test/pme/849V/status/motorMoving",10:"chng">i{1:true}
```

