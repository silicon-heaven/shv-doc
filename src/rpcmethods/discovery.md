# Discovering SHV paths and methods

SHV paths address different nodes in the SHV tree and every node has methods
associated with it.

## `*:dir`

| Name  | SHV Path | Signature    | Flags | Access |
|-------|----------|--------------|-------|--------|
| `dir` | Any      | `ret(param)` |       | Browse |

This method needs to be implemented for every node (that is every valid SHV
path). It provides a way to list all available methods and signals of the node.

| Parameter                 | Result        |
|---------------------------|---------------|
| Null \| `false` \| `true` | [i{...}, ...] |
| String                    | Bool          |

This method can be called with or without parameter. The valid parameters are:

* *Null* or `false` and in such case all methods are listed with their info.
* `true` is same as `false` or *Null* with exception that result can also
  contain extra map.
* *String* with possible method name for which `true` is returned if node has
  such method and `false` if there is no such method.

The method info in *IMap* must contain these fields:

* `1`: string containing method's name. This must be unique name for single
  node. It is not allowed to provide multiple descriptions with same name for
  the same node.
* `2`: is integer value flag assembled from the following values:
  * `1` (`1 << 0`) no longer used and reserved for compatibility reasons. In the
    past it signaled that name is not callable. New implementations should
    ignore method descriptions with this bit set.
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
  * `32` (`1 << 5`) specifies that method requires ClientID to be called.
    Calling this method without it should result in `UserIDRequired` error.
* `3`: defines parameter type for the requests. Type is a *String* identifier.
  It can be missing or have value *Null* instead of *String* if method takes no
  parameter (or only *Null*).
* `4`: defines result type for the responses. The is a *String* identifier. It
  can be missing or have value *Null* instead of *String* if method provides no
  value (or only *Null*).
* `5`: specifies minimal access level needed to call this method as *Int*. The
  allowed values can be found in table in [RpcMessage](../rpcmesasge.md)
  article.
* `6`: is used for signals associated with this method. Signals have their names
  and type identifier for value they carry. They are specified as a *Map* from
  signal's name to *String* type identifier. It is allowed to use *Null* instead
  of *String* for type and in such case type is the method's result type (of
  course field `4` must be defined).
* `63` extra *Map* that can contain anything you want. It is provided only if
  `true` is passed as argument and can be used to provide additional
  implementation specific info for this method.

Examples of `dir` requests:

```
=> <id:42, method:"dir", path:"">i{}
<= <id:42>i{2:[i{1:"dir", 2:0, 3:"idir", 4:"odir", 5:1},i{1:"ls", 2:0, 3:"ils", 4:"ols", 5:1, 6:{"lsmod":"olsmod"}}]}
```
```
=> <id:43, method:"dir", path:"test/path">i{1:false}
<= <id:43>i{2:[i{1:"dir", 2:0, 3:"idir", 4:"odir", 5:1},i{1:"ls", 2:0, 3:"ils", 4:"ols", 5:1, 6:{"lsmod":"olsmod"}},{1:"get", 2:2, 3:"iget", 4:"String", 5:8, 6:{"chng":null}}]}
```
```
=> <id:44, method:"dir", path:"test/path">i{1:"get"}
<= <id:44>i{2:true}
```
```
=> <id:44, method:"dir", path:"test/path">i{1:"nonexistent"}
<= <id:44>i{2:false}
```

The previous version (before SHV RPC 3.0) supported both *Null* and *String* but
provided *Map* instead of *IMap*. The *String* argument also always provided
list (with one or no maps). Even older implementations provided list of lists
`[[name, signature, flags, description],...]`. Clients that do want to fully
support all existing devices should support both of the old representations as
well as the latest one.

The compatibility mapping between *IMap* keys and historical *Map* is:

| IMap key | Map key                                                                           |
|----------|-----------------------------------------------------------------------------------|
| `1`      | `"name"`                                                                          |
| `2`      | `"flags"`                                                                         |
| `5`      | `"access"` or `"accessGrant"` but value is string like for `Access` in RpcMessage |

## `*:ls`

| Name | SHV Path | Signature    | Flags | Signal  | Access |
|------|----------|--------------|-------|---------|--------|
| `ls` | Any      | `ret(param)` |       | `lsmod` | Browse |

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

### Signal `lsmod`

The signal that has to be sent if there is change in the result of the `*:ls`
method. This includes case when new nodes are added as well as when nodes are
removed. The SHV Path must be to the lowest still valid node (valid after
change).

| Value         |
|---------------|
| {String:Bool} |

The value sent with notification needs to be *Map* where keys are name of the
nodes that were added or removed and value is *Bool* signaling the node
existence. `true` thus signals added node and `false` removed one. It is allowed
and desirable to specify multiple node changes in one signal but you should not
delay notification sending just to combine it with some future changes, it must
be sent as soon as possible.

```
<= <signal:"lsmod", path:"test", source:"ls">i{1:{"device":true}}
```
