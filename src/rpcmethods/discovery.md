# Discovering SHV paths and methods

SHV paths address different nodes in the SHV tree and every node has methods
associated with it.

## `*:dir`

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

## `*:ls`

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

## `*:lschng`

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
