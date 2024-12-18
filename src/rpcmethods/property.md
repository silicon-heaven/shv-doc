# Property nodes

Property node is a convention where we associate value storage with some SHV
path. The value can be received, optionally modified and its change can be
signaled.

The type of the property depends on the presence of the methods. The following
table 

|                                 | `*:get` | `*:get:*chng` | `*:set` |
|---------------------------------|---------|---------------|---------|
| Read only                       | ✔️       | ❌            | ❌      |
| Read only with signaled change  | ✔️       | ✔️             | ❌      |
| Read-write                      | ✔️       | ❌            | ✔️       |
| Read-write with signaled change | ✔️       | ✔️             | ✔️       |

## `*:get`

| Name  | SHV Path | Flags  | Param Type | Result Type | Access |
|-------|----------|--------|------------|-------------|--------|
| `get` | Any      | Getter | `i(0,)\|n` | `?`         | Read   |

This method is used for getting the current value associated with SHV path.
Every property node needs to have `get` method and every node with `get` method
can be considered as property node.

Integer argument is maximal age in milliseconds. Value can be of any age if
*Null* parameter is used (or omitted). The age is relevant when latest value
must be received over some other medium (such as from Modbus) and thus every
request would trigger a new exchange. Instead if latest exchange was withing
specified age the value can be served from cache.

```
=> <id:42, mtehod:"get", path:"test/property">i{}
<= <id:42>i{2:"hello"}
```
```
=> <id:42, method:"get", path:"test/property">i{1:60000}
<= <id:42>i{2:"Cached"}
```

## `*:get:*chng`

| Name   | Method | SHV Path | Flags  | Value Type | Access |
|--------|--------|----------|--------|------------|--------|
| `chng` | `get`  | Any      | Getter | `?`        | Read   |

Value change can be optionally signaled with signal. It is used when you have
desire to get info about value change without polling. Note that signal might
not be emitted just on value change (that means old value can be same as the new
one) and it might not be emitted on some value changes (for example if change
was under some notification deadband level). To get latest value you should
always use `*:get` instead of waiting for `*chng` signal but if you receive
`*chng` then you can save yourself a `*:get` call.

The signal name can be either just `chng` or any name with that as suffix (such
as `fchng`).

The `*chng` needs to provide the same value as `*:get` would, which is value
associated with the SHV path.

```
<= <signal:"chng", path:"test/property", source:"get">i{1:"Hello World"}
```


## `*:set`

| Name  | SHV Path | Flags  | Param Type | Result Type | Access |
|-------|----------|--------|------------|-------------|--------|
| `set` | Any      | Setter | `?`        |             | Write  |

This method is used for changing the value associated with SHV path. By
providing this method alongside with `*:get` you are making the read-write
property. Property is considered read-only if you omit this method.

The `*:get` should be providing value specified in `*:set` parameter as soon as
completion of `*:set` method is confirmed (response is sent). In other words:
the set value can be received with `*:get` right after `*:set` completes. This
rules out situation where `*:get` reports real (measured) value while `*:set`
specifies a reference. You should always split this to two property nodes where
reference is read-write property and real value is read-only one.

```
=> <id:42, method:"set", path:"test/property">i{1:"Hello World"}
<= <id:42>i{}
```
