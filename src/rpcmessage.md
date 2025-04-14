# RPC Message

Message exchanged in SHV is `IMap` with meta-data attached from
[RPC Value](rpcvalue.md) point of view.

There are three kinds of RPC messages defined:
* [Request](#request)
* [Response](#response)
* [Signal](#signal)

RPC message can have meta-data attribute defined.

| Attribute number  | Attribute name       | Type             | Description
| ----------------: | -------------------- | ---------------- | ------------
| 1                 | MetaTypeId           | Int              | Always equal to `1` in case of RPC message
| 2                 | MetaTypeNameSpaceId  | Int              | Always equal to `0` in case of RPC message, may be omitted.
| 8                 | RequestId            | Int              | Every RPC request must have unique number per client. Matching RPC response will have the same number.
| 9                 | ShvPath              | String           | Path on which method will be called.
| 10                | Method/Signal        | String           | Name of called RPC method or raised signal.
| 11                | CallerIds            | List of Int      | Internal attribute filled by broker in request message to distinguish requests with the same request ID, but issued by different clients.
| 13                | RevCallerIds         | List of Int      | Reserved for SHV v2 broker compatibility
| 14                | Access               | String           | Access granted by broker for called `shvPath` and `method` to current user. This should be used only for extra access info and for backward compatibility while `AccessLevel` is prefered instead.
| 16                | UserId               | String           | ID of user calling RPC method.
| 17                | AccessLevel          | Int              | Access level user has assigned for request or minimal access level needed to allow signal to be received.
| 18                | SeqNo                | Int              | Reserved, it will be used in next API version for multi-part messages   <https://github.com/silicon-heaven/libshv/wiki/multipart-messages>
| 19                | Source               | String           | Used for signals to store method name this signal is associated with.
| 20                | Repeat               | Bool             | Used for signals to informat that signal was emited as a repeat of some older ones (that might not might not have been sent).

Second part of RPC message is `IMap` with following possible keys.


| Key  | Key name   | Description
| ---: | ---------- | ------------
| 1    | Params     | Optional method parameters, any [RPC Value](rpcvalue.md) is allowed.
| 2    | Result     | Successful method call result, any [RPC Value](rpcvalue.md) is allowed.
| 3    | Error      | Method call exception, see [RPC error](#rpc-error) for more details
| 4    | Delay      | Method call result delay, [Double](rpcvalue.md) is allowed.
| 5    | Abort      | Method call abort (`true`) or state query (`false`), [Bool](rpcvalue.md) is allowed.

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

`userId` is string containing information about the login names and the
broker names along the RPC message path through the brokers hierarchy. The format
is`user-name1:broker-name1;user-name2:broker-name2;...`, for example:
`john@foo.bar:broker1;broker1-login:broker2`. User name and broker name is delimited by `:`,
user:broker pairs are delimited by ';'.

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

## Request

Message used to invoke remote method.

Methods invoked using this request needs to be idempotent, because RPC
transport layers do not ensure deliverability. You might also want to try to
send request again when you receive no response because of this.

Attributes:

| Attribute      | Required | Broker propagation                                            | Note                                                            |
|----------------|----------|---------------------------------------------------------------|-----------------------------------------------------------------|
| `MetaTypeId`   | yes      | copied                                                        |                                                                 |
| `RequestId`    | yes      | copied                                                        |                                                                 |
| `ShvPath`      | yes      | matched prefix removed                                        |                                                                 |
| `Method`       | yes      | copied                                                        |                                                                 |
| `CallerIds`    | no       | broker's path identifier can be added to the list             | Added and modified by brokers                                   |
| `RevCallerIds` | no       | broker's reverse path identifier can be removed from the list | If tunneling or multi-part message is needed                    |
| `Access`       | no       | set and modified                                              | Must be kept in sync with `AccessLevel` or not specified at all |
| `AccessLevel`  | no       | set and modified                                              | Broker always only reduces the already present value            |
| `UserId`       | no       | appended if present                                           | Append to non-zero string with comma                            |

Keys (only one can be used in the single message):

| Key name  | Required   | Note
| --------- | ---------- | -----
| `Params`  | no         | Any valid [RPC Value](rpcvalue.md)
| `Abort`   | yes        | [Bool](./rpcvalue.md) where `true` forces abort and `false` only immediate Reponse (thus it can be `Delay`).

**Examples**

RPC call invocation, method `switchLeft` on path `test/pme/849V` with request ID
`56` and parameter `true`. 
```
<1:1,8:56,9:"test/pme/849V",10:"switchLeft">i{1:true}
```

## Response

Response to [Request](#request). The response should be generated as soon
as possible. If some work has to be performed before full response can be
generated then [Response Delay](#responsedelay) can be sent in reasonable
intervals, until Result or Error is prepared.

Attributes:

| Attribute      | Required | Broker propagation                                                | Note                                                       |
|----------------|----------|-------------------------------------------------------------------|------------------------------------------------------------|
| `MetaTypeId`   | yes      | copied                                                            |                                                            |
| `RequestId`    | yes      | copied                                                            |                                                            |
| `CallerIds`    | no       | broker's path identifier must be removed from the list            | Sender must copy original value form *Request* if present. |
| `RevCallerIds` | no       | broker's reverse path identifier can be removed added to the list | Sender must copy original value form *Request* if present. |

Keys (only one can be used in the single message):

| Key name  | Required   | Note
| --------- | ---------- | -----
| `Result`  | no         | Used in case of successful method call result, any [RPC Value](rpcvalue.md) is allowed. It is the default if no other key is used.
| `Error`   | yes        | Required in case of method call exception, see [RPC error](#rpc-error) for more details.
| `Delay`   | yes        | Required in case nor `Result` or `Error` can't be generated immediately. The value is [Double](./rpcvalue.md) from 0 to 1.

### RPC Error

RPC Error is `IMap` with following keys defined

| Key  | Key name   | Required   | Description
| ---: | ---------- | ---------- | -------
| 1    | `Code`     | yes        | Error code
| 2    | `Message`  | no         | Error message string
| 3    | `Data`     | no         | Arbitrary payload, can be used for example for exception localization additional info.

Error codes:

| Value  | Name                          | Description
| -----: | -------                       | ----------
| 1      |                               | Reserved for backward compatibility
| 2      | `MethodNotFound`              | The method does not exist or is not available or not accessible with given access level.
| 3      | `InvalidParams`               | Invalid method parameter.
| 4      |                               | Reserved for backward compatibility
| 5      |                               | Reserved for backward compatibility
| 6      | `ImplementationReserved1`     | Won't ever be used in the communication and is reserved for implementations usage (such as signaling method timeout)
| 7      | `ImplementationReserved2`     | Same as `ImplementationReserved1`
| 8      | `MethodCallException`         | Generic execution exception of the method.
| 9      | `ImplementationReserved3`     | Same as `ImplementationReserved1`
| 10     | `LoginRequired`               | Method call without previous successful login.
| 11     | `UserIDRequired`              | Method call requires UserID to be present in the request message. Send it again with UserID.
| 12     | `NotImplemented`              | Can be used if method is valid but not implemented for what ever reason.
| 13     | `TryAgainLater`               | Used in cases when request can't be handled now but might be in the future.
| 14     | `RequestInvalid`              | Used exclusively for `Abort` request. It doesn't specify if invalidation is caused by `Abort` just that there is no such request
| 32+    | `MethodCallExceptionSpecific` | Application specific `MethodCallException`

**Examples**

Successful response to method call from example above
```
<1:1,8:56>i{2:true}
```
Exception when unknown method is called
```
<1:1,8:11>i{3:i{1:8,2:"method: foo path:  what: Method: 'foo' on path 'shv/cze' doesn't exist"}}
```

## Signal

Spontaneous message sent without prior request and thus without `RequestId`. 
It is used mainly notify clients that some technological value had changed without need to poll.

Attributes:

| Attribute     | Required | Broker propagation                     | Note                                                                             |
|---------------|----------|----------------------------------------|----------------------------------------------------------------------------------|
| `MetaTypeId`  | yes      | copied                                 |                                                                                  |
| `ShvPath`     | yes      | mouint point of the source is prefixed |                                                                                  |
| `Signal`      | no       | copied                                 | If not specified `"chng"` is assumed                                             |
| `Source`      | no       | copied                                 | If not specified `"get"` is assumed)                                             |
| `AccessLevel` | no       | copied                                 | Used to decide signal propagation on brokers, if not specified *Read* is assumed |
| `UserId`      | no       | copied                                 |                                                                                  |
| `Repeat`      | no       | copied                                 | If not specified `false` is assumed                                              |

Keys:

| Key       | Required   | Note
| --------- | ---------- | -----
| `Params`  | no         | Any valid [RPC Value](rpcvalue.md)

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
