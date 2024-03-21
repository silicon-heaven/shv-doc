# Broker API

The broker needs to implement application API, as well as some additional
methods and broker's side of the login sequence (unless it connects to other
broker and in such case it performs client side).

The broker can be pretty much ignored when you are calling methods (sending
request and receiving response messages). On the other hand, signal message
retrieval needs to be requested from broker, because broker won't pass them
automatically and thus knowledge about broker is required for clients wanting to
receive signals.

The call to `.app:ls("broker")` can be used to identify application as being a
broker.

## Signals filtering

Devices send regularly signal messages but by propagating these messages to
all clients is not always desirable. For that reason signals are filtered.
The default is to not send signal messages and clients need to explicitly
request them with subscriptions (note that it is allowed that subscriptions
would be seeded by the broker based on the login).

The speciality of these methods is that they are client specific. In general RPC
should respond to all clients the same way, if it has access rights high enough.
But these methods manage client's specific table of subscriptions, and thus it
must work with only client specific info.

Note that all the methods under `.broker/currentClient` have access set to
`Browse`. Client must always have access to these methods, so access is
irrelevant in this case. On the other hand broker must deny access to the
`.broker/currentClient:subscribe` and
`.broker/currentClient:unsubscribe` of their mount points to their clients.
That is because otherwise clients could influence subscriptions that need to be
maintained by the broker itself.

Brokers can be chained by mounting broker inside another one. Such mounted
broker of course won't automatically be sending its signal messages and thus
they won't be propagated. Simple solution is to subscribe for all signal
messages, but that is not efficient. The better solution is to derive an
appropriate subscription for mounted broker (by modifying the `"path"`) and
subscribe for only this limited set of signal messages. If you are implementing
broker then be aware that multiple subscribes can be derived to the same one and
thus unsubscribe must be performed with care to not remove still needed ones.

### `.broker/currentClient:subscribe`

| Name        | SHV Path                    | Signature     | Flags | Access |
|-------------|-----------------------------|---------------|-------|--------|
| `subscribe` | `.broker/currentClient`     | `void(param)` |       | Browse |

Adds rule that allows receiving of signals (notifications) from shv
path. The subscription applies to all methods of given name in given path and
sub-paths. The default path is an empty and thus root of the broker, this
subscribes on given method in all accessible nodes of the broker.

| Parameter | Result |
|-----------|--------|
| {...}     | Bool   |

The parameter is *Map* with:

* `"signal"` wildcard pattern (rules from POSIX.2, 3.13 with exceptions that `/`
  are not allowed) that must match the signal's name (`Signal`). It is assumed
  to be `"*"` if not specified and thus matching all names.
* `"source"` wildcard pattern (rules from POSIX.2, 3.13 with exceptions that `/`
  are not allowed) that matches the signal's source (`Source`) and thus name of
  the method this signal is associated with. It is assumed to be `"*"` if not
  specified and thus matching all names.
* `"paths"` wildcard pattern (rules from POSIX.2, 3.13 with added support for `**`
  that matches multiple nodes) that must match the signal's path (`ShvPath`). It
  is assumed to be `"**"` if not present and thus maching all nodes.
* `"methods"` an obsolete alias for `"signal"`. It is used only if `"signal"` is
  not provided. (OBSOLETE use `"signal"`)
* `"method"` with optional method name (*String*) where default is `""` and
  matches all methods. Used only if `"signal"` or `"methods"` is not provided.
  (OBSOLETE use `"methods"`)
* `"path"` with optional SHV path (*String*) where default is `""` and thus
  subscribe to all methods of given name in the tree. Used only if `"paths"` is
  not provided. (OBSOLETE use `"paths"`)

It provides `true` if sunscription was added and `false` if there was already
such subscription.

```
=> <id:42, method:"subscribe", path:".broker/currentClient">i{1:{"signal":"chng", "paths":"**"}}
<= <id:42>i{}
```
```
=> <id:42, method:"subscribe", path:".broker/currentClient">i{1:{"signal":"chng", "paths":"test/device"}}
<= <id:42>i{}
```

### `.broker/currentClient:unsubscribe`

| Name          | SHV Path                    | Signature    | Flags | Access |
|---------------|-----------------------------|--------------|-------|--------|
| `unsubscribe` | `.broker/currentClient`     | `ret(param)` |       | Browse |

Reverts an operation of `.broker/currentClient:subscribe`.

| Parameter | Result |
|-----------|--------|
| {...}     | Bool   |

The parameter must match exactly parameters used to subscribe

It provides `true` in case subscription was removed or request count modified
and `false` if it couldn't have been found.

```
=> <id:42, method:"unsubscribe", path:".broker/currentClient">i{1:{"signal":"chng"}}
<= <id:42>i{2:true}
```
```
=> <id:42, method:"unsubscribe", path:".broker/currentClient">i{1:{"signal":"chng", "paths":"invalid/**"}}
<= <id:42>i{2:false}
```

### `.broker/currentClient:subscriptions`

| Name            | SHV Path                    | Signature   | Flags  | Access |
|-----------------|-----------------------------|-------------|--------|--------|
| `subscriptions` | `.broker/currentClient`     | `ret(void)` | Getter | Browse |

This method allows you to list all existing subscriptions for the current
client.

| Parameter | Result       |
|-----------|--------------|
| Null      | [{...}, ...] |

*Map*s provided in the list will have the following fields:

* `"signal"` that is copied over from original subscription request. It is also
  used if you subscribed with `"methods"` or `"method"`.
* `"source"` that is copied over from original subscription request.
* `"paths"` that is copied over from original subscription request. It is also
  used if you subscribed with `"path"`.

```
=> <id:42, method:"subscriptions", path:".broker/currentClient">i{}
<= <id:42>i{2:[{"method":"chng"},{"signal":"chng", "paths":"test/device/**", "source":"*", "ref":1}]}
```

## Clients

The primary use for SHV Broker is the communication between multiple clients.
The information about connected clients and its parameters is beneficial not
only for the client's them self but primarily to the administrators of the SHV
network.

### `.broker:clientInfo`

| Name         | SHV Path      | Signature    | Flags | Access       |
|--------------|---------------|--------------|-------|--------------|
| `clientInfo` | `.broker` | `ret(param)`     |       | SuperService |

Information the broker has on the client.

| Parameter | Result                                                                                                                                |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------|
| Int       | {"clientId":Int, "userName":String\|Null, "mountPoint":String\|Null, "subscriptions":[i{1:String, 2:String\|Null}, ...], ...} \| Null |

The parameter is client's ID (*Int*). The provided value is *Map* with info
about the client. The *Null* is returned in case there is no client with this
ID.

#### ClientInfo
The *Map* containing at least these fields:

* `"clientId"` with *Int* containing ID assigned to this client.
* `"userName"` with *String* user name used during the login sequence. This is
  optional because login might not be always required.
* `"mountPoint"` with *String* SHV path where device is mounted. This can be
  *Null* in case this is not a device.
* `"subscriptions"` is a list of active subscriptions of this client.

Additional fields are allowed to support more complex brokers but are not
required nor standardized at the moment.

```
=> <id:42, method:"clientInfo", path:".broker">i{1:68}
<= <id:42>i{2:{"clientId:68, "userName":"smith", "subscriptions":[{1:"chng"}]}}
```
```
=> <id:42, method:"clientInfo", path:".broker">i{1:126}
<= <id:42>i{2:null}
```

### `.broker:mountedClientInfo`

| Name                | SHV Path      | Signature    | Flags | Access       |
|---------------------|---------------|--------------|-------|--------------|
| `mountedClientInfo` | `.broker` | `ret(param)`     |       | SuperService |

Information the broker has on the client that is mounted on the given SHV path.

| Parameter | Result                                                                                                                                |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------|
| String    | {"clientId":Int, "userName":String\|Null, "mountPoint":String\|Null, "subscriptions":[i{1:String, 2:String\|Null}, ...], ...} \| Null |

The parameter is SHV path (*String*). The provided value is [ClientInfo](#clientinfo) with info
about the client that is mounted on the given path (or the parent of it). The
*Null* is returned in case there is no client with such mount point.

The parameter can be either  *Int* and in such case info for client with
matching ID is provided. Or it can be *String* with SHV path for which client
mounted on the given path (this includes also all subnodes of the mount point)
is provided. The *Null* is returned in case there is no client with this ID or
path is not mounted.

The provided *Map* must contain the same fields as `.broker:clientInfo` does.

```
=> <id:42, method:"mountedClientInfo", path:".broker">i{1:"iot/device"}
<= <id:42>i{2:{"clientId:68, "userName":"smith", "subscriptions":[{1:"chng"}]}}
```
```
=> <id:42, method:"mountedClientInfo", path:".broker">i{1:"invalid"}
<= <id:42>i{2:null}
```

### `.broker/currentClient:info`

| Name   | SHV Path                    | Signature   | Flags  | Access |
|--------|-----------------------------|-------------|--------|--------|
| `info` | `.broker/currentClient`     | `ret(void)` | Getter | Browse |

Access to the information broker has for the current client. The result is
client specific.

| Parameter | Result                                                                                                                        |
|-----------|-------------------------------------------------------------------------------------------------------------------------------|
| Null      | {"clientId":Int, "userName":String\|Null, "mountPoint":String\|Null, "subscriptions":[i{1:String, 2:String\|Null}, ...], ...} |

The resulting [ClientInfo](#clientinfo) is same as for `.broker:clientInfo` called with client ID for the
current client. The difference is that this method must be accessible to the
current client while `.broker:clientInfo` is accessible only to the privileged
users.

```
=> <id:42, method:"info", path:".broker/currentClient">i{}
<= <id:42>i{2:{"clientId:68, "userName":"smith", "subscriptions":[{1:"chng"}]}}
```

### `.broker:clients`

| Name      | SHV Path      | Signature   | Flags | Access       |
|-----------|---------------|-------------|-------|--------------|
| `clients` | `.broker` | `ret(void)`     |       | SuperService |

This method allows you get list of all clients connected to the broker. This
is an administration task.

This is a mandatory way of listing clients. There also can be an optional, more
convenient way, that brokers can implement to allow easier use by administrators
(commonly in `.broker/clientInfo`), but any automatic tools should use this call
instead. It is also more efficient than using `.broker/client:ls`.

| Parameter | Result     |
|-----------|------------|
| Null      | [Int, ...] |

The *List* of *Int*s is provided where integers are client IDs of all currently
connected clients.

```
=> <id:42, method:"clients", path:".broker">i{}
<= <id:42>i{2:[68, 83]}
```
### `.broker:mounts`

| Name     | SHV Path      | Signature   | Flags | Access       |
|----------|---------------|-------------|-------|--------------|
| `mounts` | `.broker` | `ret(void)`     |       | SuperService |

This method allows you get list of all mount paths of devices connected to the broker. This
is an administration task.

| Parameter | Result        |
|-----------|---------------|
| Null      | [String, ...] |

The *List* of *Strings*s is provided where strings are mount paths of all currently mounted devices.

```
=> <id:42, method:"mounts", path:".broker">i{}
<= <id:42>i{2:["shv/device/temp1", "shv/device/temp2", ... ]}
```

### `.broker:disconnectClient`

| Name               | SHV Path      | Signature     | Flags | Access       |
|--------------------|---------------|---------------|-------|--------------|
| `disconnectClient` | `.broker` | `void(param)`     |       | SuperService |

Forces some specific client to be immediately disconnected from the SHV broker.
You need to provide client's ID as an argument. Based on the link layer client
can be immediately informed or it could just stay in the limbo until its
communication timeouts. If client is established by the broker (it is
connection to serial console or to some other broker) it is up to the broker
what should happen, but it is suggested that this would be handled as
reconnection request.

| Parameter | Result |
|-----------|--------|
| Int       | Null   |

```
=> <id:42, method:"disconnectClient", path:".broker">i{1:68}
<= <id:42>i{}
```

### `.broker/client`

It is desirable to be able to access clients directly without mounting them on a
specific path. This helps with their identification by administrators. This is
done by automatically mounting them in `.broker/client/<clientId>`. This
mount won't be reported by `.broker:mountPoints` method, nor it should be
the mount point reported in `.broker:cientInfo`. And due to not being mount
point it also can't support signal messages.

The access to this path should be allowed only to the broker administrators. The
rule of thumb is that if user can access `.broker:disconnectClient`, it should
be also able to access `.broker/client`.
