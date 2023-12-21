# Broker API

The broker needs to implement application API, as well as some additional
methods and broker's side of the login sequence (unless it connects to other
broker and in such case it performs client side).

The broker can be pretty much ignored when you are sending requests and
receiving responses to them. The notification delivery needs to be subscribed in
the broker and thus for notification the knowledge about broker is a must.

The call to `.app:ls("broker")` can be used to identify application as being a
broker.

## Notifications filtering

Devices send regularly notifications but by propagating these notifications to
all clients is not always desirable. For that reason notifications are filtered.
The default is to not send notifications and clients need to explicitly request
them with subscriptions (note that it is allowed that subscriptions would be
seeded by the broker based on the login).

The speciality of these methods is that they are client specific. In general RPC
should respond to all clients, if it has access rights high enough, the
same way. But these methods manage client's specific table of subscriptions, and
thus it must work with only client specific info.

### `.app/broker/currentClient:subscribe`

| Name        | SHV Path                    | Signature     | Flags | Access |
|-------------|-----------------------------|---------------|-------|--------|
| `subscribe` | `.app/broker/currentClient` | `void(param)` |       | Read   |

Adds rule that allows receiving of signals (notifications) from shv
path. The subscription applies to all methods of given name in given path and
sub-paths. The default path is an empty and thus root of the broker, this
subscribes on given method in all accessible nodes of the broker.

| Parameter | Result |
|-----------|--------|
| {...}     | Null   |

The parameter is *map* with:

* `"method"` with optional method name (*String*) where default is `""` and
  matches all methods.
* `"path"` with optional SHV path (*String*) where default is `""` and thus
  subscribe to all methods of given name in the tree.
* `"methods"` that replaces `"method"` and instead of being exact name of the
  method can be wildcard pattern (rules from POSIX.2, 3.13 with exceptions that
  `/` are not allowed). `"method"` even if present is ignored once this field is
  present.
* `"pattern"` that replaces `"path"` and instead of being static path it can be
  a glob wildcard pattern (rules from POSIX.2, 3.13 with added support for `**`
  that matches multiple nodes). `"path"` even if present is ignored once this
  field is present.

```
=> <id:42, method:"subscribe", path:".app/broker/currentClient">i{1:{"method":"chng"}}
<= <id:42>i{}
```
```
=> <id:42, method:"subscribe", path:".app/broker/currentClient">i{1:{"method":"chng", "path":"test/device"}}
<= <id:42>i{}
```

### `.app/broker/currentClient:unsubscribe`

| Name          | SHV Path      | Signature    | Flags | Access |
|---------------|---------------|--------------|-------|--------|
| `unsubscribe` | `.app/broker/currentClient` | `ret(param)` |       | Read   |

Reverts an operation of `.app/broker/currentClient:subscribe`. The parameter
must match exactly parameters used to subscribe.

| Parameter | Result |
|-----------|--------|
| {...}     | Bool   |

It provides `true` in case subscription was removed and `false` if it couldn't
have been found.

```
=> <id:42, method:"unsubscribe", path:".app/broker/currentClient">i{1:{"method":"chng"}}
<= <id:42>i{2:true}
```
```
=> <id:42, method:"unsubscribe", path:".app/broker/currentClient">i{1:{"method":"chng", "path":"invalid"}}
<= <id:42>i{2:false}
```

### `.app/broker/currentClient:rejectNotSubscribed`

| Name                  | SHV Path                | Signature    | Flags | Access |
|-----------------------|-------------------------|--------------|-------|--------|
| `rejectNotSubscribed` | `.app/broker/currentClient` | `ret(param)` |       | Read   |

Unsubscribes all subscriptions matching the given method and SHV path. The
intended use is when you receive notification that you are not interested in.
You can send this request with method and SHV path of such notification to just
unsubscribe. Be aware that you always subscribe on the node and all its
subnodes, you can't pick and choose nodes in the subtree with this method.

| Parameter                        | Result       |
|----------------------------------|--------------|
| {"method":String, "path":String} | [{...}, ...] |

As a result it provides subscription descriptions (see
`.app/broker/currentClient:subscribe`) that were unsubscribed.

This is primarily used for broker to broker communication. It removes need to
track subscriptions in sub-brokers (brokers mounted in the broker) because this
can be used when broker receives notification that is not subscribed by any of
its current clients and that way remove any obsolete subscription down the line.

```
=> <id:42, method:"rejectNotSubscribed", path:".app/broker/currentClient">i{1:{"method":"chng", "path":"test/device/foo"}}
<= <id:42>i{2:[{"method":"chng"}, {"method":"chng", "path":"test/device"}]}
=> <id:42, method:"rejectNotSubscribed", path:".app/broker/currentClient">i{1:{"method":"chng", "path":"test/device/foo"}}
<= <id:42>i{2:[]}
```

### `.app/broker/currentClient:subscriptions`

| Name            | SHV Path                | Signature   | Flags  | Access |
|-----------------|-------------------------|-------------|--------|--------|
| `subscriptions` | `.app/broker/currentClient` | `ret(void)` | Getter | Read   |

This method allows you to list all existing subscriptions for the current
client.

| Parameter | Result                                        |
|-----------|-----------------------------------------------|
| Null      | [{"method":String, "path":String\|Null}, ...] |

```
=> <id:42, method:"subscriptions", path:".app/broker/currentClient">i{}
<= <id:42>i{2:[{"method":"chng"},{"method":"chng", "path":"test/device"}]}
```

## Clients

The primary use for SHV Broker is the communication between multiple clients.
The information about connected clients and its parameters is beneficial not
only for the client's them self but primarily to the administrators of the SHV
network.

### `.app/broker:clientInfo`

| Name         | SHV Path  | Signature    | Flags  | Access  |
|--------------|-----------|--------------|--------|---------|
| `clientInfo` | `.app/broker` | `ret(param)` |  | SuperService |

Information the broker has on the client.

| Parameter | Result                                                                                                                                |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------|
| Int       | {"clientId":Int, "userName":String\|Null, "mountPoint":String\|Null, "subscriptions":[i{1:String, 2:String\|Null}, ...], ...} \| Null |

The parameter is client's ID (*Int*). The provided value is *Map* with info
about the client. The *Null* is returned in case there is no client with this
ID.

The provided *Map* must have at least these fields:

* `"clientId"` with *Int* containing ID assigned to this client.
* `"userName"` with *String* user name used during the login sequence. This is
  optional because login might not be always required.
* `"mountPoint"` with *String* SHV path where device is mounted. This can be
  *Null* in case this is not a device.
* `"subscriptions"` is a list of active subscriptions of this client.

Additional fields are allowed to support more complex brokers but are not
required nor standardized at the moment.

```
=> <id:42, method:"clientInfo", path:".app/broker">i{1:68}
<= <id:42>i{2:{"clientId:68, "userName":"smith", "subscriptions":[{1:"chng"}]}}
```
```
=> <id:42, method:"clientInfo", path:".app/broker">i{1:126}
<= <id:42>i{2:null}
```

### `.app/broker:mountedClientInfo`

| Name                | SHV Path  | Signature    | Flags  | Access  |
|---------------------|-----------|--------------|--------|---------|
| `mountedClientInfo` | `.app/broker` | `ret(param)` |  | SuperService |

Information the broker has on the client that is mounted on the given SHV path.

| Parameter | Result                                                                                                                                |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------|
| String    | {"clientId":Int, "userName":String\|Null, "mountPoint":String\|Null, "subscriptions":[i{1:String, 2:String\|Null}, ...], ...} \| Null |

The parameter is SHV path (*String*). The provided value is *Map* with info
about the client that is mounted on the given path (or the parent of it). The
*Null* is returned in case there is no client with such mount point.

The parameter can be either  *Int* and in such case info for client with
matching ID is provided. Or it can be *String* with SHV path for which client
mounted on the given path (this includes also all subnodes of the mount point)
is provided. The *Null* is returned in case there is no client with this ID or
path is not mounted.

The provided *Map* must contain the same fields as `.app/broker:clientInfo` does.

```
=> <id:42, method:"mountedClientInfo", path:".app/broker">i{1:"iot/device"}
<= <id:42>i{2:{"clientId:68, "userName":"smith", "subscriptions":[{1:"chng"}]}}
```
```
=> <id:42, method:"mountedClientInfo", path:".app/broker">i{1:"invalid"}
<= <id:42>i{2:null}
```

### `.app/broker/currentClient:info`

| Name   | SHV Path                | Signature   | Flags  | Access |
|--------|-------------------------|-------------|--------|--------|
| `info` | `.app/broker/currentClient` | `ret(void)` | Getter | Browse |

Access to the information broker has for the current client. The result is
client specific.

| Parameter | Result                                                                                                                        |
|-----------|-------------------------------------------------------------------------------------------------------------------------------|
| Null      | {"clientId":Int, "userName":String\|Null, "mountPoint":String\|Null, "subscriptions":[i{1:String, 2:String\|Null}, ...], ...} |

The result is same as for `.app/broker:clientInfo` called with client ID for the
current client. The difference is that this method must be accessible to the
current client while `.app/broker:clientInfo` is accessible only to the privileged
users.

```
=> <id:42, method:"info", path:".app/broker/currentClient">i{}
<= <id:42>i{2:{"clientId:68, "userName":"smith", "subscriptions":[{1:"chng"}]}}
```

### `.app/broker:clients`

| Name      | SHV Path  | Signature   | Flags  | Access  |
|-----------|-----------|-------------|--------|---------|
| `clients` | `.app/broker` | `ret(void)` |  | SuperService |

This method allows you get list of all clients connected to the broker. This
is an administration task.

This is a mandatory way of listing clients. There also can be an optional, more
convenient way, that brokers can implement to allow easier use by administrators
(commonly in `.app/broker/clientInfo`), but any automatic tools should use this call
instead. It is also more efficient than using `.app/broker/client:ls`.

| Parameter | Result     |
|-----------|------------|
| Null      | [Int, ...] |

The *List* of *Int*s is provided where integers are client IDs of all currently
connected clients.

```
=> <id:42, method:"clients", path:".app/broker">i{}
<= <id:42>i{2:[68, 83]}
```
### `.app/broker:mounts`

| Name      | SHV Path  | Signature   | Flags  | Access  |
|-----------|-----------|-------------|--------|---------|
| `mounts` | `.app/broker` | `ret(void)` |  | SuperService |

This method allows you get list of all mount paths of devices connected to the broker. This
is an administration task.

| Parameter | Result     |
|-----------|------------|
| Null      | [String, ...] |

The *List* of *Strings*s is provided where strings are mount paths of all currently mounted devices.

```
=> <id:42, method:"mounts", path:".app/broker">i{}
<= <id:42>i{2:["shv/device/temp1", "shv/device/temp2", ... ]}
```

### `.app/broker:disconnectClient`

| Name               | SHV Path  | Signature     | Flags | Access  |
|--------------------|-----------|---------------|-------|---------|
| `disconnectClient` | `.app/broker` | `void(param)` |       | SuperService |

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
=> <id:42, method:"disconnectClient", path:".app/broker">i{1:68}
<= <id:42>i{}
```

### `.app/broker/client`

It is desirable to be able to access clients directly without mounting them on a
specific path. This helps with their identification by administrators. This is
done by automatically mounting them in `.app/broker/client/<clientId>`. This mount
won't be reported by `.app/broker:mountPoints` method, nor it should be the mount
point reported in `.app/broker:cientInfo`.

The access to this path should be allowed only to the broker administrators. The
rule of thumb is that if user can access `.app/broker:disconnectClient`, it should
be also able to access `.app/broker/client`.
