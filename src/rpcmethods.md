# Common SHV RPC methods

These are well known methods used in various situations. They provide backbone
of SHV RPC communication.

## Login sequence

The login sequence needs to be performed every time a new client is connected to
the broker. It ensures that client gets correct access rights assigned and that
devices get to be mounted in the correct location in the broker's nodes tree.
Sometimes the authentication can be optional (such as when connecting over local
socket or if client certificate is used) but login sequence is required anyway
to introduce the client to the broker.

The first method to be sent by the client after connection to the broker is
established needs to be `hello` request. It is called with `null` SHV path
(which means it can be left out in the message's meta) and no parameters.

```
=> <id:1, method:"hello">i{}
```

Broker should respond with message containing nonce that can be used to perform
login. The nonce needs to be an ASCII string with length from 10 to 32
characters.

```
<= <id:1>i{2:{"nonce":"vOLJaIZOVevrDdDq"}}
```

The next message sent by the client needs to be `login` request. This message is
*Map* and needs to contain `"login"` if login is required and can contain
`"options"`.

There are two types of the logins you can use. It can be either *plain* login or
*sha1*. The difference is only the way you send the password in the message. The
*plain* login just sends password as it is in *String*. The *sha1* login hashes
the provided user's password, adds the result as suffix to the nonce from `hello`
and hashes the result again. SHA1 hash is represented in HEX format. The
complete password deduction is: `SHA1(nonce + SHA1(password))`. The SHA1 login
is highly encouraged and plain login is highly discouraged, even the low end
CPUs should be able to calculate SHA1 hash and thus perform the SHA1 login. The
`"login"` *map* needs to contain these fields:

* `"user"` with username as *String*
* `"password"` with either plain text password or SHA1 password (as described in
  the paragraph above) as *String*
* `"type"` that identifies password format, and thus it can have only one of
  these values:
  * `"PLAIN"` for *plain* text password (discouraged use)
  * `"SHA1"` for SHA1 login

The `"options"` needs to be a map of options for the broker. Broker will ignore
any unknown options. A different broker implementations can support a different
set of options, but minimal needed supported options are these:

* `"device"` with *map* containing device specific options. At least these
  options need to be supported:
  * `"deviceId"` with *String* value that identifies the device. This should be
    used by the broker to automatically assign mount point based on its internal
    rules.
  * `"mountPoint"` with *String* value that specifies path where the device's
    tree should be mounted in the broker. This should overwrite any mount point
    assigned by broker based on `"deviceId"`, but it can be disregarded if user
    doesn't have access rights for the SHV path specified.
* `"idleWatchDogTimeOut"` specifies number of seconds without message receive
  before broker will consider the connection to be dead and disconnects it. The
  default timeout by the broker should be 180 seconds. By increasing this
  timeout you can reduce the periodic dummy messages sent, but it is suggested to
  keep it in reasonable range because open but dead connection can consume
  unnecessary resources on the broker.

```
=> <id:2, method:"login">i{1:{"login":{"password":"3d613ce0c3b59a36811e4acbad533ee771afa9f3","user":"iot","type":"SHA1"}}, "options":{"device":{"deviceId":"bfsview_test", "mountPoint":"test/bfsview"}, "idleWatchDogTimeOut":180}}}
```

The broker will respond with *Null* (some older broker implementation can
respond with some value for backward compatibility reasons). From now on you can
send any requests and receive any messages you have rights on as logged user
from the broker.

```
<= <id:2>i{}
```

In case of a login error you can attempt the login again without need to
disconnect or sending `hello` again. Be aware that broker should impose delay of
60 seconds on subsequent login attempts for security reasons.

> Note that `hello` and `login` methods are not to be reported by `dir` and no
> other method can be called before login sequence is completed.


## Discovering SHV paths and methods

SHV paths address different nodes in the SHV tree and every node has methods
associated with it.

### `*:dir`

| Name  | SHV Path | Signature    | Flags | Access |
|-------|----------|--------------|-------|--------|
| `dir` | Any      | `ret(param)` |       | Browse |

This method needs to be implemented for every node (that is every valid SHV
path). It provides a way to list all available methods and signals of the node.

| Parameter       | Result                                                                                          |
|-----------------|-------------------------------------------------------------------------------------------------|
| Null \| `false` | [i{1:String, 2:Int<0,15>, 2:String, 4:String, 5:String}, ...]                                   |
| `true`          | [{"name":String, "flags":Int<0,31>, "param":String, "result":String, "access":String ...}, ...] |
| String          | i{1:String, 2:Int<0,15>, 3:String, 4:String, 5:String}                                                                                            |

This method can be called with or without parameter. The valid parameters are:

* *Null* or `false` and in such case all methods are listed with their info in
  *IMap*.
* `true` is same as `false` or *Null* with exception that result is expected to
  be *Map* instead of *IMap*. That allows custom extension that is beneficial in
  some cases. The implementations that do not support this extension are allowed
  to return *IMap* just like for *Null* or `false`.
* *String* with possible method name for which info should be provided. *Null*
  is provided if there is no such method.

The method info in both *IMap* and *Map* must contain these fields:

* `"name"` for *Map* and `1` for *IMap* with string containing method's name.
* `"flags"` for *Map* and `2` for *IMap* needs to have integer value assembled
  from the following ones:
  * `1` (`1 << 0`) specifies that method is a signal and thus can't be
    explicitly called but rather it gets automatically emitted on some event.
    Requests sent to this method will result in error about missing method.
  * `2` (`1 << 1`) specifies that method is a getter. This method must be
    callable without side effects without any parameter.
  * `4` (`1 << 2`) specifies that method is a setter. This method must be
    callable that accepts parameter and provides no value. They are commonly
    paired with getter.
  * `8` (`1 << 3`) specifies that provided value in response is going to be
    large. This exists to signal that by calling this method you can block the
    connection for considerable amount of time.
  * `16` (`1 << 4`) specifies that method is not idempotent. Such method can't
    be simply called but instead needs to be called first without parameter to
    get unique submit ID that needs to be used in argument for real call. This
    unique submit ID prevents from same request being handled multiple times
    because first execution will invalidate the submit ID and thus prevents
    re-execution.
* `"param"` for *Map* and `3` for *IMap* with *String* name of the parameter
  type as value. It can be missing or have value `null` instead of *String* if
  method takes no parameter (or `null`).
* `"result"` for *Map* and `4` for *IMap* with *String* name of the provided
  value type as value. It can be missing or have value `null` instead of
  *String* if method provides no  value (or `null`).
* `"access"` for *Map* and  `5` for *IMap* that is used to inform about minimal
  access level needed to invoke the method. If client has lower access level
  than request to this method is going to result in an error. Note that actual
  user's access level is defined by SHV broker and deduced by potentially
  complex rule set. The allowed values are:
    * `"bws"` (browse) is the lowest access level allowing to discover SHV nodes
      and methods.
    * `"rd"` (read) allows reading of values.
    * `"wr"` (write) allows setting values.
    * `"cmd"` (command) allows performing a commanding operation.
    * `"cfg"` (config) allows changing configuration values.
    * `"srv"` (service) allows access to the service info.
    * `"ssrv` (super service) allows access to the service info that can't be
      included for what ever reason in `"srv"`.
    * `"dev"` (develop) is the highest common access level. Methods with this
      access level provide access to the functionality that is needed only for
      testing and development.
    * `"su"` (admin) is the absolutely highest access level. It is commonly not
      used for the method access limitation because this level should not be
      commonly assigned. It serves as a special management access level as well
      as broker to broker level.

Examples of `dir` requests:

```
=> <id:42, method:"dir", path:"">i{}
<= <id:42>i{2:[i{1:"dir", 2:0, 3:"idir", 4:"odir", 5:"bws"},i{1:"ls", 2:0, 3:"idir", 4:"odir", 5:"bws"},{1:"lschng", 2:1, 4:"olschng", 5:"bws"}]}
```
```
=> <id:43, method:"dir", path:"test/path">i{1:false}
<= <id:43>i{2:[i{1:"dir", 2:0, 3:"idir", 4:"odir", 5:"bws"},i{1:"ls", 2:0, 3:"idir", 4:"odir", 5:"bws"},{1:"get", 2:2, 3:"iget" 4:"String", 5:"rd"}]}
```
```
=> <id:44, method:"dir", path:"test/path">i{1:"get"}
<= <id:44>i{2:i{1:"get", 2:2, 3:"iget" 4:"String", 5:"rd"}}
```
```
=> <id:44, method:"dir", path:"test/path">i{1:"nonexistent"}
<= <id:44>i{2:null}
```
```
=> <id:43, method:"dir", path:"test/path">i{1:true}
<= <id:43>i{2:[{"name":"dir", "flags":0, "param":"idir", "result":"odir", "access":"bws"},i{"name":"ls", "flags":0, "param":"idir", "result":"odir", "access":"bws"},{"name":"get", "flags":2, "param":"iget" "result":"String", "access":"rd"}]}
```

The previous version (before SHV RPC 3.0) supported both *Null* and *String* but
provided *Map* instead of *IMap*. The *String* argument also always provided
list (with one or no maps). Even older implementations provided list of lists
`[[name, signature, flags, description],...]`. Clients that do want to fully
support all existing devices should support both of the old representations as
well as the latest one.

### `*:ls`

| Name | SHV Path | Signature    | Flags | Access |
|------|----------|--------------|-------|--------|
| `ls` | Any      | `ret(param)` |       | Browse |

This method needs to be implemented for every valid SHV path. It provides a way
to list all children nodes of the node.

| Parameter | Result       |
|-----------|--------------|
| Null      | [String,...] |
| String    | Bool         |

This method can be called with or without parameter. The valid parameters are:

* *Null* and in such case all children node names are provided in the
  list of strings.
* *String* with possible child name for which `true` is returned if node has
  such child and `false` if there is no such child.

Examples of ls requests:

```
=> <id:42, method:"ls", path:"">i{}
<= <id:42>i{2:["foo", "fee", "faa"]}
```
```
=> <id:45, method:"ls", path:"">i{1:"fee"}
<= <id:45>i{2:true}
```
```
=> <id:45, method:"ls", path:"">i{1:"nonexistent"}
<= <id:45>i{2:false}
```

The previous versions (before SHV RPC 0.1) supported *Null* argument but not
*String*, instead it supported list with *String* and *Int*. This variant is
discouraged to be used by new clients. The *Null* variant is fully backward
compatible.

### `*:lschng`

| Name     | SHV Path | Signature   | Flags  | Access |
|----------|----------|-------------|--------|--------|
| `lschng` | Any      | `ret(void)` | Signal | Browse |

The signal that has to be sent if there is change in the result of the `*:ls`
method. This includes case when new nodes are added as well as when nodes are
removed.

| Value          |
|----------------|
| {String:Bool} |

The value sent with notification needs to be *Map* where keys are name of the
nodes that were added or removed and value is *Bool* signaling the node
existence. `true` thus signals added node and `false` removed one. It is allowed
and desirable to specify multiple node changes in one signal but you should not
delay notification sending just to combine it with some future changes, it must
be sent as soon as possible.

```
<= <method:"lschng", path:"test">i{1:{"device":true}}
```


## Property nodes

Property node is a convention where we associate value storage with some SHV
path. The value can be received, optionally modified and its change can be
signaled.

### `*:get`

| Name  | SHV Path | Signature    | Flags  | Access |
|-------|----------|--------------|--------|--------|
| `get` | Any      | `ret(param)` | Getter | Read   |

This method is used for getting the current value associated with SHV path.
Every property node needs to have `get` method and every node with `get` method
can be considered as property node.

| Parameter   | Result |
|-------------|--------|
| Null \| Int | Any    |

It supports an optional Integer argument which is maximal age in milliseconds.
This is used with caches along the way where sometimes `*:get` might be served
from it without need to actually address the target device.

```
=> <id:42, method:"get", path:"test/property">i{}
<= <id:42>i{2:"hello"}
```
```
=> <id:42, method:"get", path:"test/property">i{1:60000}
<= <id:42>i{2:"Cached"}
```

### `*:set`

| Name  | SHV Path | Signature     | Flags  | Access |
|-------|----------|---------------|--------|--------|
| `set` | Any      | `void(param)` | Setter | Write  |

This method is used for changing the value associated with SHV path. By
providing this method alongside with `*:get` you are making the read-write
property. Property is considered read-only if you omit this method.

The `*:get` should be providing value specified in `*:set` parameter as soon as
completion of `*:set` method is confirmed (response is sent). In other words:
the set value can be received with `*:get` right after `*:set` completes. This
rules out situation where `*:get` reports real (measured) value while `*:set`
specifies a reference. You should always split this to two property nodes where
reference is read-write property and real value is read-only one.

| Parameter | Result |
|-----------|--------|
| Any       | Null   |

```
=> <id:42, method:"set", path:"test/property">i{1:"Hello World"}
<= <id:42>i{}
```

### `*:chng`

| Name   | SHV Path | Signature   | Flags  | Access |
|--------|----------|-------------|--------|--------|
| `chng` | Any      | `ret(void)` | Signal | Read   |

This is signal, and thus it gets emitted on its own and can't be called. It is
used when you have desire to get info about value change without polling. Note
that signal might not be emitted just on value change (that means old value can
be same as the new one) and it might not be emitted on some value changes (for
example if change was under some notification dead band level). To get latest
value you should always use `*:get` instead of waiting on `*:chng` but if you
receive `*:chng` then you can safe yourself a `*:get` call.

The `*:chng` needs to provide the same value as `*:get` would, which is value
associated with the SHV path.

| Value |
|-------|
| Any   |

```
<= <method:"chng", path:"test/property">i{1:"Hello World"}
```


## Application API

These are methods that are required for every device to be present on its SHV
path `".app"`. Clients do not have to implement these, but their implementation
is highly suggested if they are supposed to be connected to the broker for more
than just a few requests.

### `.app:shvVersionMajor`

| Name              | SHV Path | Signature   | Flags  | Access |
|-------------------|----------|-------------|--------|--------|
| `shvVersionMajor` | `.app`   | `ret(void)` | Getter | Browse |

This method provides information about implemented SHV standard. Major version
number signal major changes in the standard and thus you are most likely
interested just in this number.

| Parameter | Result |
|-----------|--------|
| Null      | Int    |

```
=> <id:42, method:"shvVersionMajor", path:".app">i{}
<= <id:42>i{2:0}
```

### `.app:shvVersionMinor`

| Name              | SHV Path | Signature   | Flags  | Access |
|-------------------|----------|-------------|--------|--------|
| `shvVersionMinor` | `.app`   | `ret(void)` | Getter | Browse |

This method provides information about implemented SHV standard. Minor version
number signals new features added and thus if you wish to check for support of
these additions you can use this number.

| Parameter | Result |
|-----------|--------|
| Null      | Int    |

```
=> <id:42, method:"shvVersionMinor", path:".app">i{}
<= <id:42>i{2:1}
```

### `.app:name`

| Name      | SHV Path | Signature   | Flags  | Access |
|-----------|----------|-------------|--------|--------|
| `name` | `.app`   | `ret(void)` | Getter | Browse |

This method must provide the name of the application, or at least the SHV
implementation used in the application.

| Parameter | Result |
|-----------|--------|
| Null      | String |

```
=> <id:42, method:"name", path:".app">i{}
<= <id:42>i{2:"SomeApp"}
```

### `.app:version`

| Name         | SHV Path | Signature   | Flags  | Access |
|--------------|----------|-------------|--------|--------|
| `version` | `.app`   | `ret(void)` | Getter | Browse |

This method must provide the application version, or at least the SHV
implementation used in the application (must be consistent with information in
`.app:appName`).

```
=> <id:42, method:"version", path:".app">i{}
<= <id:42>i{2:"1.4.2-s5vehx"}
```

### `.app:ping`

| Name   | SHV Path | Signature    | Flags | Access |
|--------|----------|--------------|-------|--------|
| `ping` | `.app`   | `void(void)` |       | Browse |

This method should reliably do nothing and should always be successful. It is
used to check the connection (if message can be passed to and from client) as
well as to keep connection in case of SHV Broker.

| Parameter | Result |
|-----------|--------|
| Null      | Null   |

```
=> <id:42, method:"ping", path:".app">i{}
<= <id:42>i{}
```


## Broker API

The broker needs to implement application API, as well as some additional
methods and broker's side of the login sequence (unless it connects to other
broker and in such case it performs client side).

The broker can be pretty much ignored when you are sending requests and
receiving responses to them. The notification delivery needs to be subscribed in
the broker and thus for notification the knowledge about broker is a must.

The call to `.app:ls("broker")` can be used to identify application as being a
broker.

### Notifications filtering

Devices send regularly notifications but by propagating these notifications to
all clients is not always desirable. For that reason notifications are filtered.
The default is to not send notifications and clients need to explicitly request
them with subscriptions (note that it is allowed that subscriptions would be
seeded by the broker based on the login).

The speciality of these methods is that they are client specific. In general RPC
should respond to all clients, if it has access rights high enough, the
same way. But these methods manage client's specific table of subscriptions, and
thus it must work with only client specific info.

#### `.app/broker/currentClient:subscribe`

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

#### `.app/broker/currentClient:unsubscribe`

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

#### `.app/broker/currentClient:rejectNotSubscribed`

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

#### `.app/broker/currentClient:subscriptions`

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

### Clients

The primary use for SHV Broker is the communication between multiple clients.
The information about connected clients and its parameters is beneficial not
only for the client's them self but primarily to the administrators of the SHV
network.

#### `.app/broker:clientInfo`

| Name         | SHV Path  | Signature    | Flags  | Access  |
|--------------|-----------|--------------|--------|---------|
| `clientInfo` | `.app/broker` | `ret(param)` | Getter | Service |

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

#### `.app/broker:mountedClientInfo`

| Name                | SHV Path  | Signature    | Flags  | Access  |
|---------------------|-----------|--------------|--------|---------|
| `mountedClientInfo` | `.app/broker` | `ret(param)` | Getter | Service |

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

#### `.app/broker/currentClient:info`

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

#### `.app/broker:clients`

| Name      | SHV Path  | Signature   | Flags  | Access  |
|-----------|-----------|-------------|--------|---------|
| `clients` | `.app/broker` | `ret(void)` | Getter | Service |

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

#### `.app/broker:disconnectClient`

| Name               | SHV Path  | Signature     | Flags | Access  |
|--------------------|-----------|---------------|-------|---------|
| `disconnectClient` | `.app/broker` | `void(param)` |       | Service |

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

#### `.app/broker/client`

It is desirable to be able to access clients directly without mounting them on a
specific path. This helps with their identification by administrators. This is
done by automatically mounting them in `.app/broker/client/<clientId>`. This mount
won't be reported by `.app/broker:mountPoints` method, nor it should be the mount
point reported in `.app/broker:cientInfo`.

The access to this path should be allowed only to the broker administrators. The
rule of thumb is that if user can access `.app/broker:disconnectClient`, it should
be also able to access `.app/broker/client`.


## Device API

Device is a special application that represents a single physical device. It is
benefical to see the difference between random application and application that
runs in dedicated device and controls such device. This allows generic
identification of such devices in the SHV tree.

The call to `.app:ls("device")` can be used to identify application as being a
device.

### `.app/device:name`

| Name   | SHV Path      | Signature   | Flags  | Access |
|--------|---------------|-------------|--------|--------|
| `name` | `.app/device` | `ret(void)` | Getter | Browse |

This method must provide the device name. This is a specific generic name of the
device.

| Parameter | Result |
|-----------|--------|
| Null      | String |

```
=> <id:42, method:"name", path:".app/device">i{}
<= <id:42>i{2:"OurDevice"}
```

### `.app/device:version`

| Name      | SHV Path      | Signature   | Flags  | Access |
|-----------|---------------|-------------|--------|--------|
| `version` | `.app/device` | `ret(void)` | Getter | Browse |

This method must provide version (revision) of the device.

| Parameter | Result |
|-----------|--------|
| Null      | String |

```
=> <id:42, method:"name", path:".app/device">i{}
<= <id:42>i{2:"g2"}
```

### `.app/device:serialNumber`

| Name           | SHV Path      | Signature   | Flags  | Access |
|----------------|---------------|-------------|--------|--------|
| `serialNumber` | `.app/device` | `ret(void)` | Getter | Browse |

This method can provide serial number of the device if that is something the
device has. It is allowed to provide *Null* in case there is no serial number
assigned to this device.

| Parameter | Result         |
|-----------|----------------|
| Null      | String \| Null |

```
=> <id:42, method:"serialNumber", path:".app/device">i{}
<= <id:42>i{2:"12590"}

## SHV Journal API

### `.app/shvjournal`

Node where device history data is stored in log files. Every log file is named by timestamp of its beginning (i.e. `2023-09-11T09-00-00-000.log2`) to enable binary search in files to find correct log file.

#### Log2 format

Simple text log, every record is on single line ended with `LF`. Record fields are separated by `TAB`.

Columns:
1. `MonotonicDateTime` - monotonic timestamp of record occurrence
2. `DeviceDateTime` - this field is filled only in rare case, when device time is not equal to monotonic one, for example after restart before NTP is started
3. `ShvPath`
4. `Value` - Cpon serialized value, note that Cpon cannot contain neither `TAB` nor `LF`.
5. `ShortTime` - very simple devices without precise time can use 16-bit cyclic short-time in msec. Using short-time we can measure time interval among log records, even if we do not have correct monotonic time.
6. `Domain` - kind of record. In almost all cases is domain equal do name of SHV signal emitted when record was created. `chng` in example below means 'value changed'.
7. `ValueFlags` - record value flags bitfield
   * bit 0 - `Snapshot` - value is originated in device snapshot, it is not spontaneous
   * bit 1 - `Spontaneous` - value change is spontaneous
   * bit 2 - `DirtyValue` - reserved by history provider, not used by any device
8. `UserId` - ID of user issuing shv method call, if method calls are logged by device

example
```
cat 2023-09-11T09-00-00-000.log2

2023-09-11T09:24:28.100Z		system/plcDisconnected	false		chng	1
2023-09-11T09:24:28.100Z		system/status	0u		chng	1
2023-09-11T09:24:28.100Z		system/name	"Ell038, Vystavka"		chng	1
2023-09-11T09:24:28.100Z		system/version	"V1.00 - 202-ell038 6bbdff5e - Sep 11 2023 11:17:16"		chng	1
2023-09-11T09:24:28.100Z		system/info/tempCPU	0		chng	1
2023-09-11T09:24:28.100Z		system/info/tempPLC	0		chng	1
2023-09-11T09:24:28.100Z		system/info/memoryUserSize	0		chng	1
2023-09-11T09:24:28.100Z		system/info/memoryUserFree	0		chng	1
2023-09-11T09:24:28.100Z		system/info/memoryDRAMFree	0		chng	1
2023-09-11T09:24:28.100Z		system/ethernet/MAC	""		chng	1
2023-09-11T09:24:28.100Z		system/ethernet/IP	""		chng	1
2023-09-11T09:24:28.100Z		system/ethernet/subnetMask	""		chng	1
```
