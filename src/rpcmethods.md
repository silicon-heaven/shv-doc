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

The first method to be sent sent by the client after connection to the broker is
established needs to be `hello` request. It is called with `null` SHV path
(which means it can be left out in the message's meta) and no parameters.

```
=> <id:1, method:"hello">i{}
```

Broker should respond with message containing nonce that can be used to perform
login. The nonce needs to be a ASCII string with length from 10 to 32
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
the provided users password, adds the result as suffix to the nonce from `hello`
and hashes the result again. SHA1 hash is represented in HEX format. The
complete password deduction is: `SHA1(nonce + SHA1(password))`. The SHA1 login
is highly encouraged and plain login is highly discouraged, even the low end
CPUs should be able to calculate SHA1 hash and thus perform the SHA1 login. The
`"login"` *map* needs to contain these fields:

* `"user"` with user name as *String*
* `"password"` with either plain text password or SHA1 password (as described in
  the paragraph above) as *String*
* `"type"` that identifies password format and thus it can have only one of
  these values:
  * `"PLAIN"` for *plain* text password (discouraged use)
  * `"SHA1"` for SHA1 login

The `"options"` needs to be a map of options for the broker. Broker will ignore
any unknown options. A different broker implementations can support a different
set of options but minimal needed supported options are these:

* `"device"` with *map* containing device specific options. At least these
  otions need to be supported:
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
  timeout you can reduce the periodic dummy messages sent but it is suggested to
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

In case of an login error you can attempt the login again without need to
disconnect or sending `hello` again. Be aware that broker should impose delay of
60 seconds on subsequent login attempts for security reasons.

> Note that `hello` and `login` methods are not to be reported by `dir` and no
> other method can be called before login sequence is completed.


## Discovering SHV paths and methods

SHV paths address different nodes in the SHV tree and every node has methods
associated with it. The 

### `*:dir`

| Name  | SHV Path | Signature    | Flags | Access |
|-------|----------|--------------|-------|--------|
| `dir` | Any      | `ret(param)` |       | Browse |

This method needs to be implemented for every node (that is every valid SHV
path). It provides a way to list all available methods and signals of the node.

| Parameter | Result                                                                 |
|-----------|------------------------------------------------------------------------|
| Null      | [i{1:String, 2:Int<0,3>, 3:Int<0,15>, 4:String, 5:String\|Null}, ...]  |
| String    | i{1:String, 2:Int<0,3>, 3:Int<0,15>, 4:String, 5:String\|Null} \| Null |

This method can be called with or without parameter. The valid parameters are:

* *Null* and in such case all methods are listed
* *String* and in such case only method with matching name is listed

The provided value is list of method descriptions, or just a single method
description without list in case *String* was passed as an argument. Every
method description is *Map* with the following fields:

* `1` with string containing method's name
* `2` with integer of these possible values:
  * `0` for signature `void(void)` that is no value is returned and no parameter is
    expected.
  * `1` for signature `void(param)` that is no value is returned and some
    parameter is expected.
  * `2` for signature `ret(void)` that is value is returned and no parameter is
    expected.
  * `3` for signature `ret(param)` that is value is returned and parameter is
    expected.
* `3` that is integer assembled from the following values:
  * `1` (`1 << 0`) specifies that method is a signal and thus can't be
    explicitly called but rather it gets automatically emitted on some event. It
    is highly suggested to use this only for methods called `chng` (see the
    description about property nodes).
  * `2` (`1 << 1`) specifies that method is a getter. This method must have
    signature `ret(void)` and invoking it should have no side effect.
  * `4` (`1 << 2`) specifies that method is a setter. This method must have
    signature `void(param)`.
  * `8` (`1 << 3`) specifies that returned value is going to be large. This
    method must have signature either `ret(void)` or `ret(param)`.
* `4` that is used to inform about minimal access level needed to
  invoke the method. If client has lower access level then request to this
  method is going to result in an error. Note that actual user's access level is
  defined by SHV broker and deduced by potentially complex rule set. The allowed
  values are:
    * `"bws"` (browse) is the lowest access level allowing to discover SHV nodes
      and methods.
    * `"rd"` (read) allows reading of values.
    * `"wr"` (write) allows setting values.
    * `"cmd"` (command) allows performing a commanding operations.
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
* `5` is an optional field with string describing the method.

Examples of dir requests:

```
=> <id:42, method:"dir", path:"">i{}
<= <id:42>i{2:[i{1:"dir", 2:3, 4:"bws"},i{1:"ls", 2:3, 4:"bws"}]}
```
```
=> <id:43, method:"dir", path:"test/path">i{1:null}
<= <id:43>i{2:[i{1:"dir", 2:3, 4:"bws"},i{1:"ls", 2:3, 4:"bws"},i{1:"get", 2:3, 4:2}]}
```
```
=> <id:44, method:"dir", path:"test/path">i{1:"nonexistent"}
<= <id:44>i{2:null}
```
```
=> <id:44, method:"dir", path:"test/path">i{1:"dir"}
<= <id:43>i{2:{1:"dir", 2:3, 4:"bws"}}
```

The previous version (before SHV RPC 0.1) supported both *Null* and *String*
but they provided list with maps instead of imaps. The *string* argument also
always provided list (with one or no maps). The mapping between integer and
string keys is: `{1:"name", 2:"signature, 3:"flags", 4:"accessGrant",
5:"description"}`. Even older implementations provided list of lists (`[[name,
signature, flags, description],...]`. Clients that do want to fully support all
existing devices should support both of the old representations as well as the
latest one.

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


## Property nodes

Property node is a convention where we associate value storage with some SHV
path. The value can be received, optionally modified and its change can be
signaled.

### `*:get`

| Name  | SHV Path | Signature    | Flags  | Access |
|-------|----------|--------------|--------|--------|
| `get` | Any      | `ret(param)` | Getter | Read   |

This method is used for getting the current value associated with SHV path.
Every property node needs to have *get* method and every node with *get* method
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

This is signal and thus it gets emitted on its own and can't be called. It is
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
path `".app"`. Clients do not have to implement these but their implementation
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

### `.app:appName`

| Name      | SHV Path | Signature   | Flags  | Access |
|-----------|----------|-------------|--------|--------|
| `appName` | `.app`   | `ret(void)` | Getter | Browse |

This method must provide the name of the application, or at least the SHV
implementation used in the application.

| Parameter | Result |
|-----------|--------|
| Null      | String |

```
=> <id:42, method:"appName", path:".app">i{}
<= <id:42>i{2:"SomeApp"}
```

### `.app:appVersion`

| Name         | SHV Path | Signature   | Flags  | Access |
|--------------|----------|-------------|--------|--------|
| `appVersion` | `.app`   | `ret(void)` | Getter | Browse |

This method must provide the application version, or at least the SHV
implementation used in the application (must be consistent with information in
`.app:appName`).

```
=> <id:42, method:"appVersion", path:".app">i{}
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

### Generic

These are methods providing generic info about layout and broker functionality.

#### `.broker:mountPoints`

| Name          | SHV Path  | Signature   | Flags  | Access |
|---------------|-----------|-------------|--------|--------|
| `mountPoints` | `.broker` | `ret(void)` | Getter | Browse |

This method provides list of all mount points this broker currently serves. This
provides an easy access to the paths of all devices. Broker should filter this
list based on the access rights of the client requesting this.

| Parameter | Result        |
|-----------|---------------|
| Null      | [String, ...] |

```
=> <id:42, method:"mountPoints", path:".broker">i{}
<= <id:42>i{2:["test/device", "test/foo", "test/site/device"]}
```

#### `.broker/currentClient:mountPoint`

| Name         | SHV Path                | Signature   | Flags  | Access |
|--------------|-------------------------|-------------|--------|--------|
| `mountPoint` | `.broker/currentClient` | `ret(void)` | Getter | Browse |

Getter for the mount point of the current client. The result is client specific.
Broker can assign any mount point to the device based on its rules and this is
the way client can identify where it ended up in the broker's tree.

| Parameter | Result |
|-----------|--------|
| Null      | String |

```
=> <id:42, method:"mountPoint", path:".broker/currentClient">i{}
<= <id:42>i{2:"test/device"}
```

#### `.broker/app:ping`

| Name   | SHV Path      | Flags | Access |
|--------|---------------|-------|--------|
| `ping` | `.broker/app` |       | Browse |

This is alias for `.app:ping`. This one must be provided for historical purposes
(compatibility with clients before SHV 0.1).  The `.app:ping` should be
preferred for the new clients.

| Parameter | Result |
|-----------|--------|
| Null      | Null   |

### Notifications filtering

Devices send regularly notifications but by propagating these notification to
all clients is not always desirable. For that reason notifications are filtered.
The default is to not send notifications and clients need to explicitly request
them with subscriptions (note that it is allowed that subscriptions would be
seeded by the broker based on the login).

The speciality of these methods is that they are client specific. In general RPC
should respond to all clients, if it they have high enough access rights, the
same way. But these methods manage client's specific table of subscriptions and
thus it must work with only client specific info.

#### `.broker/app:subscribe`

| Name        | SHV Path      | Signature     | Flags | Access |
|-------------|---------------|---------------|-------|--------|
| `subscribe` | `.broker/app` | `void(param)` |       | Browse |

Adds rule that allows receive of change notifications from method and optional
path. The subscription applies to all methods of given name in given path or
sub-path. The default path is an empty and thus root of the broker, this
subscribes on given method in all accessible nodes of the broker.

| Parameter                              | Result |
|----------------------------------------|--------|
| {"method":String, "path":String\|Null} | Null   |

The parameter is *map* with:

* `"method"` with method name (*String*)
* `"path"` with optional SHV path (*String*) where default is `""` and thus
  subscribe to all methods of given name in the tree.

```
=> <id:42, method:"subscribe", path:".broker">i{1:{"method":"chng"}}
<= <id:42>i{}
```
```
=> <id:42, method:"subscribe", path:".broker">i{1:{"method":"chng", "path":"test/device"}}
<= <id:42>i{}
```

#### `.broker/app:unsubscribe`

| Name          | SHV Path      | Signature    | Flags | Access |
|---------------|---------------|--------------|-------|--------|
| `unsubscribe` | `.broker/app` | `ret(param)` |       | Browse |

Reverts an operation of `.broker/app:subscribe`. The parameter must match
exactly parameters used to subscribe.

| Parameter                              | Result |
|----------------------------------------|--------|
| {"method":String, "path":String\|Null} | Bool   |

It provides `true` in case subscription was removed and `false` if it couldn't
have been found.

```
=> <id:42, method:"unsubscribe", path:".broker">i{1:{"method":"chng"}}
<= <id:42>i{2:true}
```
```
=> <id:42, method:"unsubscribe", path:".broker">i{1:{"method":"chng", "path":"invalid"}}
<= <id:42>i{2:false}
```

#### `.broker/app:rejectNotSubscribed`

| Name                  | SHV Path      | Signature    | Flags | Access |
|-----------------------|---------------|--------------|-------|--------|
| `rejectNotSubscribed` | `.broker/app` | `ret(param)` |       | Browse |

Unsubscribes all subscriptions matching the given method and SHV path.  The
intended use is when you receive notification that you are not interested in.
You can send this request with method and SHV path of such notification to just
unsubscribe. Be aware that you always subscribe on the node and all its
subnodes, you can't pick and choose nodes in the subtree with this method.

| Parameter                        | Result                                        |
|----------------------------------|-----------------------------------------------|
| {"method":String, "path":String} | [{"method":String, "path":String\|Null}, ...] |

As a result it provides subscription descriptions that were unsubscribed.

This is primarily used for broker to broker communication. It removes need to
track subscriptions in sub-brokers (brokers mounted in the broker) because this
can be used when broker receives notification that is not subscribed by any of
its current clients and that way remove any obsolete subscription down the line.

```
=> <id:42, method:"rejectNotSubscribed", path:".broker">i{1:{"method":"chng", "path":"test/device/foo"}}
<= <id:42>i{2:[{"method":"chng"}, {"method":"chng", "path":"test/device"}]}
=> <id:42, method:"rejectNotSubscribed", path:".broker">i{1:{"method":"chng", "path":"test/device/foo"}}
<= <id:42>i{2:[]}
```

#### `.broker/currentClient:subscriptions`

| Name            | SHV Path      | Signature   | Flags  | Access |
|-----------------|---------------|-------------|--------|--------|
| `subscriptions` | `.broker/app` | `ret(void)` | Getter | Browse |

This method allows you to list all existing subscriptions for the current
client.

| Parameter | Result                                        |
|-----------|-----------------------------------------------|
| Null      | [{"method":String, "path":String\|Null}, ...] |

```
=> <id:42, method:"subscriptions", path:".broker">i{}
<= <id:42>i{2:[{"method":"chng"},{"method":"chng", "path":"test/device"}]}
```

### Clients

The primary use for SHV Broker is the communication between multiple clients.
The information about connected clients and its parameters is beneficial not
only for the client's them self but primarily to the administrators of the SHV
network.

#### `.broker:clientInfo`

| Name         | SHV Path  | Signature    | Flags  | Access |
|--------------|-----------|--------------|--------|--------|
| `clientInfo` | `.broker` | `ret(param)` | Getter | Browse |

Information the broker has for the current client. This info about itself should
be accessible to the client but using any other client ID should require access
level *Service*.

| Parameter   | Result                                                                                                                                |
|-------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Null \| Int | {"clientID":Int, "userName":String\|Null, "mountPoint":String\|Null, "subscriptions":[i{1:String, 2:String\|Null}, ...], ...} \| Null |

The parameter can be either *Null* (no parameter), and in such case the user info
is for the current client. Or it can be *Int* and in such case info is for
client with matching ID. The *Null* is returned in case there is no client with
this ID.

The provided *Map* must have at least these fields:

* `"clientID"` with *Int* containing ID assigned to this client.
* `"userName"` with *String* user name used during the login sequence. This is
  optional because login might not be always required.
* `"mountPoint"` with *String* SHV path where device is mounted. This can be
  *Null* in case this is not a device.
* `"subscriptions"` is a list of active subscriptions of this client.

Additional fields are allowed to support more complex brokers but are not
required nor standardized at the moment.

```
=> <id:42, method:"clientID", path:".broker">i{}
<= <id:42>i{2:{"clientID:68, "userName":"smith", "subscriptions":[{1:"chng"}]}}
```
```
=> <id:42, method:"clientID", path:".broker">i{1:68}
<= <id:42>i{2:{"clientID:68, "userName":"smith", "subscriptions":[{1:"chng"}]}}
```
```
=> <id:42, method:"clientID", path:".broker">i{1:126}
<= <id:42>i{2:null}
```

#### `.broker:clients`

| Name      | SHV Path  | Signature   | Flags  | Access  |
|-----------|-----------|-------------|--------|---------|
| `clients` | `.broker` | `ret(void)` | Getter | Service |

This method allows you get info about all clients connected to the broker. This
is an administration task.

This is mandatory way of listing clients. There is also can be an optional more
convenient way that brokers can implement to allow easier use by administrators
(commonly in `.broker/clients`), but any automatic tools should use this call
instead.

| Parameter | Result                                                                                                                               |
|-----------|--------------------------------------------------------------------------------------------------------------------------------------|
| Null      | [{"clientID":Int, "userName":String\|Null, "mountPoint":String\|Null, "subscriptions":[i{1:String, 2:String\|Null}, ...], ...}, ...] |

The *List* of maps is provided. The content is the same as `.broker:clientInfo`
provides.

```
=> <id:42, method:"clients", path:".broker">i{}
<= <id:42>i{2:[{"clientID:68, "userName":"smith", "subscriptions":[{1:"chng"}]]}, {"clientID":83, "userName":"iot", "mountPoint":"iot/device"}]}
```

#### `.broker:disconnectClient`

| Name               | SHV Path  | Signature     | Flags | Access  |
|--------------------|-----------|---------------|-------|---------|
| `disconnectClient` | `.broker` | `void(param)` |       | Service |

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
=> <id:42, method:"clients", path:".broker">i{1:68}
<= <id:42>i{}
```

#### `.broker/currentClient:clientId`

| Name       | SHV Path                | Signature   | Flags  | Access |
|------------|-------------------------|-------------|--------|--------|
| `clientId` | `.broker/currentClient` | `ret(void)` | Getter | Browse |

Access to the identifier used by the broker to identify the current client. The
result is client specific.

| Parameter | Result |
|-----------|--------|
| Null       | Int   |

```
=> <id:42, method:"clientId", path:".broker/currentClient">i{}
<= <id:42>i{2:68}
```

#### `.broker/clientAccess`

It is desirable to be able to access clients directly without mounting them on a
specific path. This helps with their identification by administrators. This is
done by automatically mounting them in `.broker/clientAccess/<clientID>`. This
mount won't be reported by `.broker:mountPoints` method nor it should be the
mount point reported by `.broker/currentClient:mountPoint`.

The access to this path should be allowed only to the broker administrators. The
rule of thumb is that if user can access `.broker:disconnectClient`, it should
be also able to access `.broker/clientAccess`.
