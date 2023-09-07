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
  * `255` - all attributes as `imap`
  * `6` - bitfield of attributes enum numbers, return only `Name` and `Signature` attribute values as `imap`, consider `enum {Name = 1, Signature, AccessGrant, Description, ...}`
     

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
  * `255` - all attributes as `imap`
  * `6` - bitfield of attributes enum numbers, return only `Name` and `HasChildren` attribute values as `imap`, consider `enum {Name = 1, HasChildren}`
