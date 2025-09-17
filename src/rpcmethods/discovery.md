# Discovering SHV paths and methods

SHV paths address different nodes in the SHV tree and every node has methods
associated with it.

## `*:dir`

| Name  | SHV Path | Flags | Param Type | Result Type | Access |
|-------|----------|-------|------------|-------------|--------|
| `dir` | Any      |       | `n\|b\|s`  | `[!dir]\|b` | Browse |

This method needs to be implemented for every node (that is every valid SHV
path). It provides a way to list all available methods and signals of the node.

Be aware that for existing nodes the result of this method should be constant.
There should be no methods added nor removed during the node existence.

This method has three variants of the behavior invoked based on the parameter.
The valid parameters and their behavior are:

* *Null* or `false` and in such case all methods are listed with their info.
* `true` is same as `false` or *Null* with exception that result can also
  contain extra map.
* *String* with possible method name for which `false` is provided if there is
  no such method and some other value (except of `null`) when there is.

The method info in *IMap* must contain these fields:

* `1` (*name*): string containing method's name. This must be unique name for
  single node. It is not allowed to provide multiple descriptions with same name
  for the same node.
* `2` (*flags*): is integer value flag assembled from the following values:
  * `1` (`1 << 0`) no longer used and reserved for compatibility reasons. In the
    past it signaled that name is not callable. New implementations should
    ignore method descriptions with this bit set.
  * `2` (*isGetter*, `1 << 1`) specifies that method is a getter. This method
    must be callable without side effects without any parameter.
  * `4` (*isSetter*, `1 << 2`) specifies that method is a setter. The usage of
    this flag is deprecated and should no longer be used.
  * `8` (*largeResult*, `1 << 3`) specifies that provided value in response is
    going to be large. This exists to signal that by calling this method you can
    block the connection for considerable amount of time.
  * `16` (*notIdempotent*, `1 << 4`) specifies that method is not idempotent.
    Such method can't be simply called but instead needs to be called first
    without parameter to get unique submit ID that needs to be used in argument
    for real call. This unique submit ID prevents from same request being
    handled multiple times because first execution will invalidate the submit ID
    and thus prevents re-execution.
  * `32` (*userIDRequired*, `1 << 5`) specifies that method requires UserID to
    be called. Calling this method without it should result in `UserIDRequired`
    error.
  * `64` (*isUpdatable*, `1 << 6`) specifies that value received as parameter
    will be extended by some already present values. This extension is defined
    for compound types (List, Map, IMap) as addition of fields not present in
    the parameter. Non-compound types do not allow items omission and thus this
    flag has no meaning with them. The usage is expected on setters with known
    associated getter such as in case of [property nodes](./property.md).
  * `128` (*longExecution*, `1 << 7`): The method execution can be longer than
    normal request-respond exchange. The caller should avoid opportunistic
    calls.
* `3` (*paramType*): defines parameter type for the requests. Type is a *String*
  that must adhere to [type description definition](../rpctypes.md). It can be
  missing or have value *Null* instead of *String* if method takes no parameter
  (which is same as taking parameter *Null*).
* `4` (*resultType*): defines result type for the responses. The is a *String*
  that must adhere to [type description definition](../rpctypes.md). It can be
  missing or have value *Null* instead of *String* if method provides no value
  (which is same as providing *Null* result).
* `5` (*accessLevel*): specifies minimal access level needed to call this method
  as *Int* (in range from 0 to 63). The defined values can be found in table in
  [RpcMessage](../rpcmessage.md) article.
* `6` (*signals*): is used for signals associated with this method. Signals have
  their names and type identifier for value they carry. They are specified as a
  *Map* from signal's name to *String* type identifier. It is allowed to use
  *Null* instead of *String* for type and in such case type is the method's
  result type (of
  course field `4` must be defined).
* `63` (*extra*): extra *Map* that can contain anything you want. It is provided
  only if `true` is passed as argument and can be used to provide additional
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
| `3`      | `"param"`                                                                         |
| `4`      | `"result"`                                                                        |
| `5`      | `"access"` or `"accessGrant"` but value is string like for `Access` in RpcMessage |
| `6`      | `"signals"`                                                                        |
| `63`     | `"tags"`                                                                        |

## `*:ls`

| Name | SHV Path | Flags | Param Type | Result Type | Access |
|------|----------|-------|------------|-------------|--------|
| `ls` | Any      |       | `s\|n`     | `[s]\|b`    | Browse |

This method needs to be implemented for every valid SHV path. It provides a way
to list all children nodes of the node.

This method has two variants of the behavior based on the parameter. The valid
parameters and their behavior are:

* *Null* and in such case all children node names are provided in the
  list of strings.
* *String* with possible child name for which `false` is provided if node has
  no such child and some other value (except of `null`) if there is.

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

### `*:ls:lsmod`

| Name    |      | SHV Path | Flags | Value Type | Access |
|---------|------|----------|-------|------------|--------|
| `lsmod` | `ls` | Any      |       | `{b}`      | Browse |

The signal that has to be sent if there is change in the result of the `*:ls`
method. This includes case when new nodes are added as well as when nodes are
removed. The SHV Path must be to the lowest still valid node (valid after
change).

The value sent with notification needs to be *Map* where keys are name of the
nodes that were added or removed and value is *Bool* signaling the node
existence. `true` thus signals added node and `false` removed one. It is allowed
and desirable to specify multiple node changes in one signal but you should not
delay notification sending just to combine it with some future changes, it must
be sent as soon as possible.

```
<= <signal:"lsmod", path:"test", source:"ls">i{1:{"device":true}}
```
