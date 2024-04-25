# RpcMessage

`RpcMessage` is `IMap` with meta-data attached from [RpcValue](rpcvalue.md) point of view.

There are three kinds of RPC messages defined:
* [RpcRequest](#rpcrequest)
* [RpcResponse](#rpcresponse)
* [RpcSignal](#rpcsignal)

RPC message can have meta-data attribute defined.

| Attribute number  | Attribute name       | Type             | Description
| ----------------: | -------------------- | ---------------- | ------------
| 1                 | MetaTypeId           | Int              | Always equal to `1` in case of RPC message
| 2                 | MetaTypeNameSpaceId  | Int              | Always equal to `0` in case of RPC message, may be omitted.
| 8                 | RequestId            | Int              | Every RPC request must have unique number per client. Matching RPC response will have the same number.
| 9                 | ShvPath              | String           | Path on which method will be called.
| 10                | Method/Signal        | String           | Name of called RPC method or raised signal.
| 11                | CallerIds            | List of Int      | Internal attribute filled by broker in request message to distinguish requests with the same request ID, but issued by different clients.
| 13                | RespCallerIds        | List of Int      | Reserved, internal attribute filled by broker in response message to enable support for multi-part messages and tunneling. <https://github.com/silicon-heaven/libshv/wiki/multipart-messages>
| 14                | Access               | String           | Access granted by broker for called `shvPath` and `method` to current user. This should be used only for extra access info and for backward compatibility while `AccessLevel` is prefered instead.
| 16                | UserId               | String           | ID of user calling RPC method.
| 17                | AccessLevel          | Int              | Access level user has assigned for request or minimal access level needed to allow signal to be received.
| 18                | Part                 | Bool             | Reserved, it will be used in next API version for multi-part messages   <https://github.com/silicon-heaven/libshv/wiki/multipart-messages>
| 19                | Source               | String           | Used for signals to store method name this signal is associated with.
| 20                | Repeat               | Bool             | Used for signals to informat that signal was emited as a repeat of some older ones (that might not might not have been sent).

Second part of RPC message is `IMap` with following possible keys.


| Key  | Key name   | Description
| ---: | ---------- | ------------
| 1    | Params     | Optional method parameters, any [RpcValue](rpcvalue.md) is allowed.
| 2    | Result     | Successful method call result, any [RpcValue](rpcvalue.md) is allowed.
| 3    | Error      | Method call exception, see [RPC error](#rpc-error) for more details

`RequestId` can be any unique number assigned by side that sends request
initially. It is used to pair up requests with their responses. The common
approach is to just use request message counter as request ID.

`CallerIds` are related to the broker and unless you are implementing a
broker you need to know only that they are additional identifiers for the
message alongside the `RequestId` to pair requests with their responses and thus
should be always included in the response message if they were present in the
request.

The `ShvPath` is used to select exact node of method in the SHV tree. 

`AccessLevel` is the way to specify access level. It is numerical with
predefined range (0-63) and brokers on the way can lower this number to even
further limit access. Broker are prohibited to increase this number. `Access`
should be used to get level if this field is not present. *Admin* access level
should be considered as the base limit if neither `AccessLevel` nor `Access`
field is present.

`Access` is older approach for the access control. It is assigned to request by
broker according to user rights. Multiple grants can be specified and separated
by comma. This is no longer the primary way and is used only for pre-SHV 3.0
peers or if additional access fields are specified.

The following are all defined `AccessLevel`s and their `Access` representations:

| Name          | Numerical representation | `Access` representation |
|---------------|--------------------------|-------------------------|
| Browse        | 1                        | `bws`                   |
| Read          | 8                        | `rd`                    |
| Write         | 16                       | `wr`                    |
| Command       | 24                       | `cmd`                   |
| Config        | 32                       | `cfg`                   |
| Service       | 40                       | `srv`                   |
| Super-service | 48                       | `ssrv`                  |
| Development   | 56                       | `dev`                   |
| Admin         | 63                       | `su`                    |

## RpcRequest

Message used to invoke remote method.

Methods invoked using this request needs to be idempotent, because RPC
transport layers do not ensure deliverability. You might also want to try to
send request again when you receive no response because of this.

Attributes

| Attribute      | Required | Broker propagation                                            | Note                                                            |
|----------------|----------|---------------------------------------------------------------|-----------------------------------------------------------------|
| `MetaTypeId`   | yes      | copied                                                        |                                                                 |
| `RequestId`    | yes      | copied                                                        |                                                                 |
| `ShvPath`      | yes      | matched prefix removed                                        |                                                                 |
| `Method`       | yes      | copied                                                        |                                                                 |
| `RevCallerIds` | no       | broker's reverse path identifier can be removed from the list | If tunneling or multi-part message is needed                    |
| `CallerIds`    | no       | broker's path identifier can be added to the list             | Added and modified by brokers                                   |
| `Access`       | no       | set and modified                                              | Must be kept in sync with `AccessLevel` or not specified at all |
| `AccessLevel`  | no       | set and modified                                              | Broker always only reduces the already present value            |
| `UserId`       | no       | appended if present                                           | Append to non-zero string with comma                            |

Keys

| Key       | Required   | Note
| --------- | ---------- | -----
| `Params`  | no         | Any valid [RpcValue](rpcvalue.md)

**Examples**

RPC call invocation, method `switchLeft` on 
path `test/pme/849V` with request ID `56` and parameter `true`. 
```
<1:1,8:56,9:"test/pme/849V",10:"switchLeft">i{1:true}
```

## RpcResponse

Response to [RpcRequest](rpcrequest.md)

Attributes

| Attribute      | Required | Broker propagation                                                | Note                                                          |
|----------------|----------|-------------------------------------------------------------------|---------------------------------------------------------------|
| `MetaTypeId`   | yes      | copied                                                            |                                                               |
| `RequestId`    | yes      | copied                                                            |                                                               |
| `RevCallerIds` | no       | broker's reverse path identifier can be removed added to the list | Sender must copy original value form *RpcRequest* if present. |
| `CallerIds`    | no       | broker's path identifier can be removed from the list             | Sender must copy original value form *RpcRequest* if present. |

Keys

| Key       | Required   | Note
| --------- | ---------- | -----
| `Result`  | yes        | Required in case of successful method call result, any [RpcValue](rpcvalue.md) is allowed.
| `Error`   | yes        | Required in case of method call exception, see [RPC error](#rpc-error) for more details.

### RPC Error

RPC Error is `IMap` with following keys defined

| Key  | Key name   | Required   | Description
| ---: | ---------- | ---------- | -------
| 1    | `Code`     | yes        | Error code
| 2    | `Message`  | no         | Error message string
| 3    | `Data`     | no         | Arbitrary payload, can be used for example for exception localization aditional info.

Error codes

| Value  | Name                  | Description
| -----: | -------               | ----------
| 1      | `InvalidRequest`      | The `RpcValue` sent is not a valid RPC Request object.
| 2      | `MethodNotFound`      | The method does not exist or is not available or not accessible with given access level.
| 3      | `InvalidParams`       | Invalid method parameter.
| 4      | `InternalError`       | Internal RPC error.
| 5      | `ParseError`          | Invalid `ChainPack` was received.
| 6      | `MethodCallTimeout`   | Method execution timed out (for example if method implementation needs more time to access some other resources), you should try call later.
| 7      | `MethodCallCancelled` |
| 8      | `MethodCallException` | Generic execution exception of the method.
| 9      | `Unknown`             | Used if error failure can't be determined (use only if absolutely necessary)
| 10     | `LoginRequired`       | Method call without previous successful login.
| 11     | `UserIDRequired`      | Method call requires UserID to be present in the request message. Send it again with UserID.
| 12     | `NotImplemented`      | Can be used if method is valid but not implemented for what ever reason.
| 32     | `UserCode`            |

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

| Attribute     | Required | Broker propagation                     | Note                                                                             |
|---------------|----------|----------------------------------------|----------------------------------------------------------------------------------|
| `MetaTypeId`  | yes      | copied                                 |                                                                                  |
| `ShvPath`     | yes      | mouint point of the source is prefixed |                                                                                  |
| `Signal`      | no       | copied                                 | If not specified `"chng"` is assumed                                             |
| `Source`      | no       | copied                                 | If not specified `"get"` is assumed)                                             |
| `AccessLevel` | no       | copied                                 | Used to decide signal propagation on brokers, if not specified *Read* is assumed |
| `UserId`      | no       | copied                                 |                                                                                  |
| `Repeat`      | no       | copied                                 | If not specified `false` is assumed                                              |

Keys

| Key       | Required   | Note
| --------- | ---------- | -----
| `Params`  | no         | Any valid [RpcValue](rpcvalue.md)

Warning: all signal names ending with `chng` (that includes `chng` and others
such as `fchng`) are considered as property changes of the method they are
associated with, and thus cache can use to update its state them instead of
method call. If your method emits signals with different parameter than
response's result then do not use signal names ending with `chng`.

**Examples**

Spontaneous notification about change of `status/motorMoving` property to `true`.
```
<1:1,9:"shv/test/pme/849V/status/motorMoving",10:"chng",11:"get">i{1:true}
```
