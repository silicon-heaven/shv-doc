## SHV RPC defined methods

## hello

TBD

## login

TBD

## dir

`dir(filter, attributes)`

List node methods

parameters
* `filter`
  * `null` - all methods 
  * `"foo"` - method `foo` only
* `attributes`
  * `null` - no attributes 
  * `[]` - all attributes as `map`
  * `["namme", "signature"]` - return only `name` and `signature` attribute values as `map`
  * `0` - all attributes as `imap`
  * `6` - bitfield of attributes enum numbers, return only `Name` and `Signature` attribute values as `imap`, consider `enum {Name = 1, Signature, Flags, AccessGrant, Description, Tags}`

examples

`dir()`
```
[
  "dir",
  "ls"
]
```
`dir([null, []])`
```
[
  {"accessGrant":"bws", "flags":0u, "name":"dir", "signature":3},
  {"accessGrant":"bws", "flags":0u, "name":"ls", "signature":3}
]
```
`dir(["ls", []])`
```
[
  i{4:"bws", 3:0u, 1:"ls", 2:3}
]
```

## ls

`ls(filter = null, attributes = null)`

List node children

parameters
* `filter`
  * `null` - all nodes 
  * `"foo"` - nodes`foo` only
* `attributes`
  * `null` - no attributes 
  * `[]` - all attributes as `map`
  * `["namme", "hasChildren"]` - return only `name` and `hasChildren` attribute values as `map`
  * `0` - all attributes as `imap`
  * `6` - bitfield of attributes enum numbers, return only `Name` and `HasChildren` attribute values as `imap`, consider `enum {Name = 1, HasChildren}`

examples

`ls()`
```
[
  "foo",
  "bar"
]
```
`ls([null, []])`
```
[
  {"hasChildren":false, "name":"foo"},
  {"hasChildren":true, "name":"bar"},
]
```
`ls(["bar", []])`
```
[
  {2:true, 1:"bar"},
]
```