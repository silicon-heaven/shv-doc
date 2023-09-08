# Common SHV RPC methods

These are well known methods of set of them used in various situations.

## Login sequence

The login sequence needs to be performed every time a new client is connected to
the broker. It ensures that client gets correct access rights assigned and that
devices get to be mounted in the correct location in the broker's nodes tree.
Sometimes the authentication can be optional (such as when connecting over local
socket or if client certificate is used) but login sequence is required anyway
to introduce the client to the broker.

The first method to be sent sent by the client after connection to the broker is
established needs to be `hello` request. It is called with `null` SHV path and
no parameters.

```
<id:1, method:"hello">i{}
```

Broker should respond with message containing nonce that can be used to perform
login. The nonce needs to be a ASCII string with length from 10 to 32
characters.

```
<id:1>i{2:{"nonce":"vOLJaIZOVevrDdDq"}}
```

The next message sent by the client needs to be `login` request. This message
needs to contain login if login is required and can contain options.

> Note that `hello` and `login` methods are not to be reported by `dir` and no
> other method can be called before login sequence is completed.


## Discovering SHV paths and methods

SHV paths address different nodes in the SHV tree and every node has methods
associated with it. The 

### dir

> Name: `"dir"` | Signature: `ret(param)` | Browse access

This method needs to be implemented for every node (that is every valid SHV
path). It provides a way to list all available methods and signals of the node.

This method can be called with or without parameter. The valid parameters are:

* *Null* and in such case all methods are listed
* *String* and in such case only method with matching name is listed

The returned value is list of method descriptions, or just a single method
description without list in case *String* was passed as an argument. Every
method description is *Map* with the following fields:

* `name` with string containing method's name
* `signature` with integer of these possible values:
  * `0` for signature `void(void)` that is no value is returned and no parameter is
    expected.
  * `1` for signature `void(param)` that is no value is returned and some
    parameter is expected.
  * `2` for signature `ret(void)` that is value is returned and no parameter is
    expected.
  * `3` for signature `ret(param)` that is value is returned and parameter is
    expected.
* `flags` that is integer assembled from the following values:
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
* `accessGrant` that is used to inform about minimal access level needed to
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
* `description` is an optional field with string describing the method.

Examples of dir requests:

```
=> <id:42, method:"dir", path:"">i{}
<= <id:42>i{2:[{"name":"dir", "signature":3, "accessGrant":"bws"}]}
```
```
=> <id:43, method:"dir", path:"test/path">i{1:null}
<= <id:43>i{2:[{"name":"dir", "signature":3, "accessGrant":"bws"},{"name":"ls", "signature":3, "accessGrant":"bws"}]}
```
```
=> <id:44, method:"dir", path:"test/path">i{1:"nonexistent"}
<= <id:44>i{2:null}
```
```
=> <id:44, method:"dir", path:"test/path">i{1:"dir"}
<= <id:43>i{2:{"name":"dir", "signature":3, "accessGrant":"bws"}}
```

In addition to the 

### ls

> Name: `"ls"` | Signature: `ret(param)` | Browse access

This method needs to be implemented for every valid SHV path. It provides a way
to list all children nodes of the node.

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

The previous versions (before SHV RPC 0.1.0) supported *Null* argument but not
*String*, instead it supported list with *String* and *Int*. This variant is
discouraged to be used by new clients. The *Null* variant is fully backward
compatible.


## Property nodes

Property node is a convention where we associate value storage with some SHV
path. The value can be received, optionally modified and its change can be
signaled.

### get

> Name: `"get"` | Signature: `ret(void)` | Getter | Read access

This method is used for getting the current value associated with SHV path.
Every property node needs to have *get* method and every node with *get* method
can be considered as property node.

```
=> <id:42, method:"get", path:"test/property">i{}
<= <id:42>i{2:"hello"}
```

### set

> Name: `"set"` | Signature: `void(param)` | Setter | Write access

This method is used for changing the value associated with SHV path. By
providing this method alongside with `get` you are making the read-write
property. Property is considered read-only if you omit this method.

The `get` should be providing value specified in `set` parameter as soon as
completion of `set` method is confirmed (response is sent). In other words: the
set value can be received with `get` right after `set` completes. This rules out
situation where `get` reports real (measured) value while `set` specifies a
reference. You should always split this to two property nodes where reference is
read-write property and real value is read-only one.

```
=> <id:42, method:"set", path:"test/property">i{1:"Hello World"}
<= <id:42>i{}
```

### chng

> Name: `"chng"` | Signature: `ret(void)` | Signal | Read access

This is signal and thus it gets emitted on its own and can't be called. It is
used when you have desire to get info about value change without polling. Note
that signal might not be emitted just on value change (that means old value can
be same as the new one) and it might not be emitted on some value changes (for
example if change was under some notification dead band level). To get latest
value you should always use `get` instead of waiting on `chng` but if you
receive `chng` then you can safe yourself a `get` call.

The `chng` needs to provide the same value as `get` would which is value
associated with the SHV path.

```
<= <method:"chng", path:"test/property">i{1:"Hello World"}
```


## Application identification

These are methods that are required for every device to be present on the SHV
path `".app"`. Clients do not have to implement these but their implementation
is highly suggested if they are suppose to be connected to the broker for more
than just few requests. They are used to differentiate between different clients
connected to the broker.

### shvVersion

> Name: `"shvVersion"` | SHV path: `".app"` | Signature: `ret()` | Getter | Browse access

### appName

> Name: `"appName"` | SHV path: `".app"` | Signature: `ret()` | Getter | Browse access

This method should be present on SHV path `".app"`.

### appVersion

> Name: `"appVersion"` | SHV path: `".app"` | Signature: `ret()` | Getter | Browse access

### echo

> Name: `"echo"` | SHV path: `".app"` | Signature: `ret(param)` | Getter | Browse access


## SHV Broker API

### .broker/app

#### subscribe

#### unsubscribe

#### rejectNotSubscribed

### .broker/currentClient

#### clientId

#### mountPoint
