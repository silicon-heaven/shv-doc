# RpcMessage

`RpcMessage` is `IMap` with meta-data attached from [RpcValue](rpcvalue.md) point of view.

There are three kinds of RPC message defined:
* [RpcRequest](#rpcrequest)
* [RpcResponse](#rpcresponse)
* [RpcSignal](#rpcsignal)

RPC message can have meta-data attribute defined.

Attribute number | Attrinbute name | Description
----------------:|-----------------|------------
1                | MetaTypeId          | Always equal to `1` in case of RPC message 
2                | MetaTypeNameSpaceId | Always equal to `0` in case of RPC message, may be omitted.
8                | RequestId           | Every RPC request must have unique number per client. Matching RPC response will have the same number. 
9                | ShvPath             | Path on which method will be called. 
10               | Method              | Name of called RPC method 
11               | CallerIds           | Internal attribute filled by broker to distinguish request from different clients with the same request ID.  
13               | RevCallerIds        | Internal attribude filled by broker to enable support for multi-part messages and tunelling.
14               | AccessGrant         | Acess granted by broker to called `shvPath` and `method` to current user. 
16               | UserId              | ID of usser calling RPC method filled in by broker.

Second part of RPC message is `IMap` with following possible keys.


Key | Key name | Description
---:|----------|------------
1   | Params   | Optional method parameters, any [RpcValue](rpcvalue.md) is allowed.
2   | Result   | Successful method call result, any [RpcValue](rpcvalue.md) is allowed.
3   | Error    | Method call exception, see [RPC error](#rpc-error) fo more details

Let describe three types of messages from example above in more details.

## RpcRequest

Message used to invoke remote method.

Attributes

Attribute | Required | Note
----------|----------|-----
`MetaTypeId`   | yes | Always set to `1`
`RequestId`    | yes |
`ShvPath`      | yes |
`Method`       | yes |
`RevCallerIds` | no  | If tunelling or multi-part message is needed
`CallerIds`    |     | Attributes managed by broker
`AccessGrant`  |     | Attributes managed by broker
`UserId`       |     | Attributes managed by broker

Keys

Key      | Required | Note
---------|----------|-----
`Params` | no       | Any valid [RpcValue](rpcvalue.md)

RPC call invocation,  method `switchLeft` on 
path `test/pme/849V` with request ID `56` and parameter `true`. 
```
<1:1,8:56,9:"test/pme/849V",10:"switchLeft">i{1:true}
```

## RpcResponse

Attributes

Attribute | Required | Note
----------|----------|-----
`MetaTypeId`   | yes | Always set to `1`
`RequestId`    | yes | ID of matching `RpcRequest`
`RevCallerIds` | yes | Must be copied from `RpcRequest` if present.
`CallerIds`    | yes | Must be copied from `RpcRequest` if present.

Keys

Key      | Required | Note
---------|----------|-----
`Result` | yes      | Required in case of successful method call result, any [RpcValue](rpcvalue.md) is allowed.
`Error`  | yes      | Required in case of method call exception, see [RPC error](#rpc-error) fo more details.

### RPC Error

RPC Error is `IMap` with following keys defined

Key | Key name  | Required | Description
---:|---------- |----------|-------
1   | `Code`    | yes      | Error code
2   | `Message` | no       | Optional message
3   | `MsgVals` | no       | List of values needed to localize `Message` on caller side.

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
It is used mainly notify clients that some tochological value had changed withou need to poll.

Attributes

Attribute | Required | Note
----------|----------|-----
`MetaTypeId`   | yes | Always set to `1`
`ShvPath`      | yes | Property path
`Method`       | yes | Signal name, the property changes have obviously method `chng`

Keys

Key      | Required | Note
---------|----------|-----
`Params` | no       | Any valid [RpcValue](rpcvalue.md)

Spontaneous notification about change of `status/motorMoving` property to `true`.
```
<1:1,9:"shv/test/pme/849V/status/motorMoving",10:"chng">i{1:true}
```

