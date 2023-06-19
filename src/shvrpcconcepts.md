# SHV RPC concepts

## Messages in the network

There are three types of messages used in the Silicon Heaven RPC communication.

[RpcRequest](rpcrequest.md) -
message used to call some method. Such messages is required to have some
request ID, SHV path, method name and optionally some parameters

[RpcResponse](rpcresponse.md) -
message sent by side that received some request. This message has to have
request ID and result but it can't have method name. 

[RpcSignal](rpcsignal.md) -
message sent without prior request and thus without request ID. It needs to
specify SHV path and method name. It also can contain parameters with any valid [RpcValue](rpcvalue.md).

SHV RPC network design
----------------------

Silicon Heaven RPC is point to point protocol. There are always only two sides
in the communication.  Broadcast is not utilized.

Messages are transmitted between two sides while both sides can send requests.

The bigger networks are constructed using brokers. Broker supports multiple
client connections in parallel and provides exchange of messages between them.

It is a good practice to always connect sides through broker and never directly.
That is because broker is the only one that needs to manage multiple connections
in parallel in such case. Clients are only connected to the broker.

Clients can be simple dummy clients that only call some methods and listen for
responses and signals but there are also clients that expose methods and nodes.
We call these clients "devices". Device has tree of nodes where every node can
have multiple methods associated with it. This tree can be discovered through
method `ls` that needs to be implemented on every node. Methods associated
with some node can be discovered with `dir` method that also has to be
implemented on every node.

Device's tree needs to be accessible to clients through SHV broker and this is
archived by attaching it somewhere in the broker's tree. This operation is
called mounting and you need to specify mount point (node in the SHV broker
tree) when connecting device to the SHV broker. Thanks to this client can
communicate with multiple devices in parallel as well as device can be used by
multiple clients.


SHV RPC Broker
--------------

Broker is an element in the network that allows exchange of the messages between
multiple clients. To connected clients it behaves like a device with exception
that some SHV paths are not handled directly by it but rather propagated to some
other client. The message propagation depends on its type.

`RpcRequest` - 
Broker looks up the correct client it should forward message to
based on the SHV path. It handles request itself if there is no such client.
If client is located, broker modifies SHV path by removing mount point of the
client, by adding caller's ID (which is client ID of the original client) and
by assigning granted access level. The message is then sent to the located
client. Broker doesn't remember this request because that is not required.
`RpcResponse` is returned to the correct client based on the caller's ID.

As stated, target client is located based on some mount path. This mount path
is assigned by broker `mounts` table according to client `device.id` provided
by client on login. Client can also choose its own mount path under directory
`test` for testing or development purposes. In that case, the login option
`device.mountPoint` is used.

`RpcSignal` -
Signals are propagated based on the subscriptions clients made
beforehand. All clients are checked for the subscription and if message matches
some then it is propagated to that client. The SHV path of the message is
prefixed with mount point of the client the signal was received from. This way
other clients see signals as being delivered from the correct place in the SHV
tree.

Clients can subscribe by calling method `subscribe` on path `.broker/app`.
This method expects map with keys `path` and `method`. Subscribe applies to
any node that is under given path and method that matches specified method name.
Empty method subscription behaves like wild card for any method.

The previous subscription can be canceled with method `unsubscribe` on path
`.broker/app`. The parameters need to be the same as for previous
`subscribe`. It returns `true` when such subscribe is located and `false`
otherwise.

There is also special method `rejectNotSubscribed` on path `.broker/app`
that allows users to cancel subscription without knowing the exact parameters
used to subscribe. You need to provide `path` and `method` and it locates
first matching subscription and cancels it. `true` is returned if that is
successful and `false` otherwise. Note that by sending empty message until you
get `false` you can unsubscribe all subscriptions.


Access control
--------------

User can be limited from accessing some methods. The right of access is
controlled by the device that handles request not by the intermediate brokers.
At the same time devices don't and should not know about user accounts and thus
the complete access control is in reality split to two steps. Client sends
request to the broker and it assigns to the message some access grants based on
its own rules. The message is then delivered to the device that checks this
grants and either performs the method or raises error based on it.

The predefined access grants are the following:

Grant name | Description
-----------|------------
`bws`      | is the lowest possible access grant. This level allows user to list SHV nodes tree and to discover methods. Nothing more is allowed.
`rd`       | provides user with read access and thus access should be allowed only to methods that perform reading of values. Those method should not have side effects.
`wr`       | provides user with write access and thus access should be allowed to the method that modify some values.
`cmd`      | provides user with access to methods that control and command the device.
`cfg`      | provides user with access to methods used to modify device's configuration.
`srv`      | provides user with access to methods used to service devices and SHV network.
`ssrv`     | provides user with access to methods used to service devices and SHV network that can harm the network or device.
`dev`      | provides user with access to methods used only for development purposes.
`su`       | provides user with access to all methods. It has also unique feature that it keeps message access grant as received. This makes it the level you want to use to include broker in other broker (chaining brokers).

Levels are sorted from the lowest to the highest and are understood to include
all lover level rights.

There might more access grant assigned by broker separated by coma `,`. Every device should ignore unknown grants and proceed or refuse
method call according to known ones.
