# SHV RPC concepts

SHV RPC is a protocol that allows remote control, data collection and general
supervision on multiple devices. It tries to land middle ground between "feature
rich" and "minmal and simple". It archives it by breaking different
responsibilities to a different components. The full network is built from these
components.

## Messages in the network

There are three types of messages used in the Silicon Heaven RPC communication.
Together they facilitate two types of communication:

* Remote procedure call (referenced as method call in SHV RPC)
* Signal broadcasting

Remote procedure calls are always confirmed, while signals are never confirmed.

[RpcRequest](rpcmessage.md#rpcrequest) - message used to call some method. Such
messages are required to exactly identify such method and can optionally carry
parameters for it. They also must specify request ID that is later used to pair
response message with request.

[RpcResponse](rpcmessage.md#rpcresponse) - message sent by side that received
some request. This message copies request ID from request message to allow
calling peer to identify it as the response for that specific request.

[RpcSignal](rpcmessage.md#rpcsignal) - spontaneous message sent without prior
request and thus without request ID. Optionally it can carry parameter.

## SHV RPC network design

Silicon Heaven RPC is point to point protocol. There are always only two sides
in the communication. Broadcast is not directly utilized (do not confuse this
with signal broadcasting that is performed by SHV RPC Broker and described in
next chapter).

Messages are transmitted between two sides. Both sides can send both requests
and well as signals and they should respond to requests with responses.

The bigger networks are constructed using brokers. Broker supports multiple
client connections in parallel and provides exchange of messages between them.

It is a good practice to always connect sides through broker and never directly.
That is because broker is the only one that needs to manage multiple connections
in parallel in such case. Clients are only connected to the broker.

Clients can be simple dummy clients that only call some methods and listen for
responses and signals, but there are also clients that expose methods and nodes.
These are mountable clients. Mountable clients has tree of nodes where every
node can have multiple methods associated with it. This tree can be discovered
through method `ls` that needs to be implemented on every node. Methods
associated with some node can be discovered with `dir` method that also has to
be implemented on every node.

Mountable client's tree needs to be accessible to other clients through SHV
broker and this is archived by attaching it somewhere in the broker's tree. This
operation is called mounting, and you need to specify mount point (node in the
SHV broker's tree) when connecting client to the SHV broker. Thanks to this
clients can communicate with multiple mounted clients in parallel as well as
mounted client can be used by multiple clients.


## SHV RPC Broker

Broker is an element in the network that allows exchange of the messages between
multiple clients. To connected clients it behaves like a mountable client with
exception that some SHV paths are not handled directly by it but rather
propagated to some other client. The message propagation depends on its type.

[RpcRequest](rpcmessage.md#rpcrequest) - Broker looks up the correct mounted
client it should forward message to based on the SHV path. It handles request
itself if there is no such client. If mounted client is located, broker removes
client's mount point from request's SHV path, it appends client ID of caller to
`CallerIds`, and assigns granted access level based on its configuration. The
message is then sent to the located client. Broker doesn't remember this request
because all the request state is contained in request meta-data.

[RpcResponse](rpcmessage.md#rpcresponse) is returned to the correct client based
on the `CallerIds` contained in request. Note, that client must copy `RequestId`
and `CallerIds` from request to response meta-data, if it should be delivered
back to caller.

[RpcSignal](rpcmessage.md#rpcsignal) - Signals are propagated based on the
subscriptions clients made beforehand. All clients are checked for the
subscription and if message matches some, then it is propagated to that client.
The SHV path of the message is prefixed with mount point of the client the
signal was received from. This way other clients see signals as being delivered
from the correct place in the SHV tree. Signals received from not-mounted
clients are simply dropped.

The API client's have available from Broker is documented in [Broker
API](./rpcmethods/broker.md).

Brokers must maintain their own subscriptions in mounted clients that are broker
so they receive signals and can propagate them further.


## Access control

User can be limited from accessing some methods. The right of access is
controlled by the client that handles request not by the intermediate brokers.
At the same time clients don't and should not know about user accounts and thus
the complete access control is in reality split to two steps. Client sends
request to the broker, and it assigns to the message some access level based on
users rules. The message is then delivered to the mounted client that checks
this level and either performs the method or raises error based on it. Brokers
can choose to handle request with error if not even minimal browse access is
given to the client.

The predefined access levels are the following:

| Name          | Description                                                                                                                                                  |
|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Browse        | The lowest possible access level. This level allows user to list SHV nodes and to discover methods. Nothing more is allowed.                                 |
| Read          | Provides user with read access and thus access should be allowed only to methods that perform reading of values. Those methods should not have side effects. |
| Write         | Provides user with write access and thus access should be allowed to the method that modify some values.                                                     |
| Command       | Provides user with access to methods that control and command.                                                                                               |
| Config        | Provides user with access to methods used to modify configuration.                                                                                           |
| Service       | Provides user with access to methods used to service devices and SHV network.                                                                                |
| Super-service | Provides user with access to methods used to service devices and SHV network that can harm the network or device.                                            |
| Development   | Provides user with access to methods used only for development purposes.                                                                                     |
| Admin         | Provides user with access to all methods.                                                                                                                    |

Levels are sorted from the lowest to the highest and are understood to include
all lover level rights.
