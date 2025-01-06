# Broker API

The broker needs to implement application API, as well as some additional
methods and broker's side of the login sequence (unless it connects to other
broker and in such case it performs client side).

The broker can be pretty much ignored when you are calling methods (sending
request and receiving response messages). On the other hand, signal message
retrieval needs to be requested from broker, because broker won't pass them
automatically and thus knowledge about broker is required for clients wanting to
receive signals.

The call to `:ls(".broker")` can be used to identify application as being a
broker.

## Signals filtering

Devices send regularly signal messages but by propagating these messages to all
clients is not always desirable. For that reason signals are filtered. The
default is to not send signal messages and clients need to explicitly request
them with subscriptions (note that it is allowed that subscriptions would be
seeded by the broker based on the login).

The speciality of these methods is that they are client specific. In general RPC
should respond to all clients the same way, if it has access rights high enough,
but these methods manage client's specific table of subscriptions, and thus it
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
appropriate subscription for mounted broker and subscribe for only this limited
set of signal messages. If you are implementing broker then be aware that
multiple subscribes can be derived to the same one and thus unsubscribe must be
performed with care to not remove still needed ones. The solution for that is to
not use unsubscribe and rely on TTL instead.

### `.broker/currentClient:subscribe`

| Name        | SHV Path                | Flags | Param Type           | Result Type | Access |
|-------------|-------------------------|-------|----------------------|-------------|--------|
| `subscribe` | `.broker/currentClient` |       | `s\|[s:RPCRI,i:TTL]` | `b`         | Browse |

Adds rule that allows receiving of signals (notifications) from shv
path. The subscription applies to all methods of given name in given path and
sub-paths. The default path is an empty and thus root of the broker, this
subscribes on given method in all accessible nodes of the broker.

The parameter is [resource identifier for signals](../rpcri.md). There is also
an option to subscribe only for limited time by using list parameter where first
argument is RPC RI and the second is TTL in seconds. The subscribe with
specified TTL is automatically dropped when given number of seconds elapses.
This time can be extended by calling `subscribe` with TTL again or it can be
even removed when called without TTL.

It provides `true` if subscription was added and `false` if there was already
such subscription.

```
=> <id:42, method:"subscribe", path:".broker/currentClient">i{1:"**:*:chng"}
<= <id:42>i{2:true}
```
```
=> <id:42, method:"subscribe", path:".broker/currentClient">i{1:["test/device/**:get:chng", 120]}
<= <id:42>i{2:true}
```

### `.broker/currentClient:unsubscribe`

| Name          | SHV Path                | Flags | Param Type | Result Type | Access |
|---------------|-------------------------|-------|------------|-------------|--------|
| `unsubscribe` | `.broker/currentClient` |       | `s`        | `b`         | Browse |

Reverts an operation of `.broker/currentClient:subscribe`.

The parameter must be [resource identifier for signal](../rpcri.md) used for
subscription creation.

It provides `true` in case subscription was removed and `false` if it couldn't
have been found.

```
=> <id:42, method:"unsubscribe", path:".broker/currentClient">i{1:"**:*:chng"}
<= <id:42>i{2:true}
```
```
=> <id:42, method:"unsubscribe", path:".broker/currentClient">i{1:"invalid/**:*:chng"}
<= <id:42>i{2:false}
```

### `.broker/currentClient:subscriptions`

| Name            | SHV Path                | Flags  | Param Type | Result Type | Access |
|-----------------|-------------------------|--------|------------|-------------|--------|
| `subscriptions` | `.broker/currentClient` | Getter |            | `{i\|n}`       | Browse |

This method allows you to list all existing subscriptions for the current
client.

Map of strings to int key value pairs is provided where keys are [resource
identifiers for signals](../rpcri.md) and values are TTL remaining for the
existing subscriptions. *Null* TTL means, that the subscription lasts forever. 

```
=> <id:42, method:"subscriptions", path:".broker/currentClient">i{}
<= <id:42>i{2:{"**:*:chng":null, "test/device/**:get:chng":739}}
```

## Clients

The primary use for SHV Broker is the communication between multiple clients.
The information about connected clients and its parameters is beneficial not
only for the client's them self but primarily to the administrators of the SHV
network.

### `.broker:clientInfo`

| Name         | SHV Path  | Flags | Param Type | Result Type | Access       |
|--------------|-----------|-------|------------|-------------|--------------|
| `clientInfo` | `.broker` |       | `i`        | `{?}\|n`    | SuperService |

Information the broker has on the client.

The parameter is client's ID (*Int*). The provided value is *Map* with info
about the client. The *Null* is provided in case there is no client with this
ID.

The *Map* containing at least these fields:

* `"clientId"` (`i`) with *Int* containing ID assigned to this client.
* `"userName"` (`s|n`) with *String* user name used during the login sequence.
  This can be *Null* because broker can have clients it established itself and
  thus won't perform any login.
* `"mountPoint"` (`s|n`) with *String* SHV path where device is mounted. This
  can be *Null* in case this is not a device.
* `"subscriptions"` (`{i|n}`) is a *Map* of subscriptions of this client. It is
  same as `.broker/currentClient:subscriptions` provides.

Additional fields are allowed to support more complex brokers but are not
required nor standardized at the moment.

```
=> <id:42, method:"clientInfo", path:".broker">i{1:68}
<= <id:42>i{2:{"clientId:68, "userName":"smith", "mountPoint": "iot/device", "subscriptions":{"**:*:chng":null}}}
```
```
=> <id:42, method:"clientInfo", path:".broker">i{1:126}
<= <id:42>i{2:null}
```

### `.broker:mountedClientInfo`

| Name                | SHV Path  | Flags | Param Type | Result Type | Access       |
|---------------------|-----------|-------|------------|-------------|--------------|
| `mountedClientInfo` | `.broker` |       | `s`        | `{?}\|n`    | SuperService |

Information the broker has on the client that is mounted on the given SHV path.

The parameter is SHV path (*String*). The provided value is
same as [`.broker:clientInfo`](#brokerclientinfo) with info about the client
that is mounted on the given path (or the parent of it). The *Null* is provided
in case there is no mount point to which given path would belong to.

```
=> <id:42, method:"mountedClientInfo", path:".broker">i{1:"iot/device/node"}
<= <id:42>i{2:{"clientId:68, "userName":"smith", "mountPoint": "iot/device", "subscriptions":{"**:*:chng":null}}}
```
```
=> <id:42, method:"mountedClientInfo", path:".broker">i{1:"invalid"}
<= <id:42>i{2:null}
```

### `.broker/currentClient:info`

| Name   | SHV Path                | Flags  | Param Type | Result Type | Access |
|--------|-------------------------|--------|------------|-------------|--------|
| `info` | `.broker/currentClient` | Getter |            | `{?}\|n`    | Browse |

Access to the information broker has for the current client. The result is
client specific.

The provided value is same as if [`.broker:clientInfo`](#brokerclientinfo) would
be called with client ID for the current client. The difference is that this
method must be accessible to the current client while `.broker:clientInfo` is
accessible only to the privileged users.

```
=> <id:42, method:"info", path:".broker/currentClient">i{}
<= <id:42>i{2:{"clientId:68, "userName":"smith", "mountPoint": "iot/device", "subscriptions":[{1:"chng"}]}}
```

### `.broker:clients`

| Name      | SHV Path  | Flags | Param Type | Result Type | Access       |
|-----------|-----------|-------|------------|-------------|--------------|
| `clients` | `.broker` |       |            | `[i]`       | SuperService |

This method allows you get list of all clients connected to the broker. This
is an administration task.

This is a mandatory way of listing clients. There also can be an optional, more
convenient way, that brokers can implement to allow easier use by administrators
(commonly in `.broker/clientInfo`), but any automatic tools should use this call
instead. It is also more efficient than using `.broker/client:ls`.

The *List* of *Int*s is provided where integers are client IDs of all currently
connected clients.

```
=> <id:42, method:"clients", path:".broker">i{}
<= <id:42>i{2:[68, 83]}
```
### `.broker:mounts`

| Name     | SHV Path  | Flags | Param Type | Result Type | Access       |
|----------|-----------|-------|------------|-------------|--------------|
| `mounts` | `.broker` |       |            | `[s]`       | SuperService |

This method allows you get list of all mount paths of devices connected to the
broker. This is an administration task.

The *List* of *Strings*s is provided where strings are mount points of all
currently mounted clients.

```
=> <id:42, method:"mounts", path:".broker">i{}
<= <id:42>i{2:["iot/device", "shv/device/temp1"]}
```

### `.broker:disconnectClient`

| Name               | SHV Path  | Flags | Param Type | Result Type | Access       |
|--------------------|-----------|-------|------------|-------------|--------------|
| `disconnectClient` | `.broker` |       | `i`        |             | SuperService |

Forces some specific client to be immediately disconnected from the SHV broker.
You need to provide client's ID as an argument. If client is established by the
broker (it is connection to serial console or to some other broker) it is up to
the broker what should happen, but it is suggested that this would be handled as
reconnection request.

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
