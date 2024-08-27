# Bytes Exchange node

| ❗ This document is in DRAFT stage |
|------------------------------------|

SHV RPC is by design request-response communication protocol and most notably
messages can be dropped without a retransmit nor notice and thus lost. At the
same time there are use cases when it is desirable to emulate byte streams such
as remote terminal. This requires improved reliability in the form of ensured
delivery and both sides transmission control.

In the RPC communication there are always two sides, the asking peer (caller)
and the answering peer (answerer). To get the bidirectional communication we
exchange bytes between caller and answerer with method call. This gives full
control over the communication flow to the caller that must initiate every
exchange. The answerer holds the connection status and notifies caller on
status change with SVH signal, but it doesn't send any to be exchanged bytes on
its own.

The communication is initialized by requesting the new exchange point by caller
from answerer. This exchange point is then used to exchange bytes between them.
It is deleted either by either sides closing it by an appropriate action or if
caller doesn't call `exchange` for `idleTimeOut` (in default 30 seconds).

## `*:newExchange`

| Name          | SHV Path | Flags | Access          |
|---------------|----------|-------|-----------------|
| `newExchange` | Any      |       | Write or higher |

This creates new connection for this bytes exchange node. The
connections are maintained as sub-nodes of this one.

The access level should be at minimum write but it might be increased depending
on the resource this provides. For example you really do not want to provide
access to root shell with just write access level.

| Parameter | Result |
|-----------|--------|
| {...}     | String |

The parameter is Map with options to be set to the new exchange point. The real
options depend on the implementation. The minimum supported list is described in
`*/ASSIGNED:options`.

The result of this call is name of the created node. The names of the sub-nodes
is up to the implementation and its algorithm for generating them but it should
choose short names. The soft limit for the name is 8 characters which gives
versatility for assignment for node name generation but still limits length of
the name that clients can expect to remember. For security reasons the
assignment should minimize the reuse of the node names as much as possible.

```
=> <id:42, method:"newExchange", path:"test/sh">i{}
<= <signal:"lsmod", path:"test/sh", source:"ls">i{1:{"a3":true}}
<= <id:42>i{2:"a3"}
```

## `*/ASSIGNED:exchange`

| Name       | SHV Path                   | Flags | Access |
|------------|----------------------------|-------|--------|
| `exchange` | Bellow Bytes Exchange node |       | Write  |

This is the method that is called to exchange bytes.

The access level must be disregarded for this method and instead CallerIds must
be used to control access. The message must have same CallerIds as the one for
`*:newExchange` that created this node. Nobody else must be allowed to call this
method (this is for consistency of exchanged bytes).

| Parameter                 | Result                    |
|---------------------------|---------------------------|
| i{0:UInt, 1:UInt: 3:Blob} | i{1:UInt, 2:UInt, 3:Blob} |

The parameter is IMap with following items:

| Key | Name           | Type | Description                                                                                                                                                                    |
|-----|----------------|------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 0   | Counter        | UInt | The counter for exchanges. This counts from 0 to 63 and wraps back to zero. It is used to detect multiple call attempts in case response got lost. This field must be present. |
| 1   | ReadyToReceive | UInt | Number of bytes caller is ready to receive in the response. The default if not specified is `0`.                                                                               |
| 3   | Data           | Blob | Bytes from caller. It doesn't have to be present if no data is being send.                                                                                                     |

The result is IMap with following items:

| Key | Name           | Type | Description                                                                                           |
|-----|----------------|------|-------------------------------------------------------------------------------------------------------|
| 1   | ReadyToReceive | UInt | Number of bytes answerer is ready to receive in the next exchange. The default if not present is `0`. |
| 2   | ReadyToSend    | UInt | Number of bytes answerer is ready to send in the next exchange. The default if not present is `0`.    |
| 3   | Data           | Blob | Bytes from answerer. It doesn't have to be present if no data is being send.                          |

Note that ReadyToSend is only informative. The implementation of answerer might
not know number of available bytes and this most likely it will use only `0` for
"no bytes available" and `1` for "some bytes available". The parameter's
ReadyToReceive thus should always contain maximum number of bytes caller is
willing to receive and not copy of ReadyToSend from previous result or signal.

```
=> <id:42, method:"exchange", path:"test/sh/a3">i{0:7, 1:32, 3:b"Foo"}
<= <id:42>i{2:i{1:24, 3:b"Hello Foo\n"}}
```

## `*/ASSIGNED:exchange:ready`

The caller is expected to call `*/ASSIGNED:exchange` as soon as it can again to
send and/or receive more data, but in some cases it might want to just wait for
data to become available or space available for receive. In such a situation
caller must poll `*/ASSIGNED:exchange` periodically to detect that, but that
introduces a time delay. The solution is that answerer sends this signal when
either ReadyToReceive or ReadyToSend goes from zero to non-zero value. The
caller still must poll the exchange method to not get stuck in case signal gets
lost but in most cases this signal will speed up the return to the data
exchanging again.

| Value             |
|-------------------|
| i{1:UInt, 2:UInt} |

The value is IMap with items ReadyToReceive and ReadyToSend from
`*/ASSIGNED:exchange` result.

```
<= <signal:"ready", path:"test/sh/a3", source:"exchange">i{1:i{1:1}}
```

The more realistic exchange to the one used as an example for
`*/ASSIGNED:exchange` would contain this signal in the following way:

```
=> <id:42, method:"exchange", path:"test/sh/a3">i{0:7, 1:32, 3:b"Foo"}
<= <id:42>i{2:i{1:24}}
<= <signal:"ready", path:"test/sh/a3", source:"exchange">i{1:i{1:32, 2:1}}
=> <id:43, method:"exchange", path:"test/sh/a3">i{0:8, 1:32}
<= <id:43>i{2:i{1:32, 3:b"Hello Foo\n"}}
```

## `*/ASSIGNED:options`

| Name      | SHV Path                   | Flags  | Access        |
|-----------|----------------------------|--------|---------------|
| `options` | Bellow Bytes Exchange node | Getter | Super-service |

This provide access to the options associated with this connection.

The access level applies only to the callers that do not match CallerIds.
Request messages with CallerIds matching that for `*:newExchange` that created
this node must be allowed.

| Parameter | Result |
|-----------|--------|
| Null      | {...}  |

The result is Map with at least these fields:

* `idleTimeOut` with unsigned integer with number of seconds. It allows client
  to specify its own idle time out. If not specified the 30 seconds are used.

```
=> <id:42, method:"options", path:"test/sh/a3">i{}
<= <id:42>i{2:{"idleTimeOut":30}}
```

## `*/ASSIGNED:setOptions`

| Name         | SHV Path                   | Flags  | Access        |
|--------------|----------------------------|--------|---------------|
| `setOptions` | Bellow Bytes Exchange node | Setter | Super-service |

This allows modification of option associated with this connection. Note that
not all options might be modifiable and only fields specified in Map are
modified (the rest is kept same or derived from new options based on the
implementation).

The access level applies only to the callers that do not match CallerIds.
Request messages with CallerIds matching that for `*:newExchange` that created
this node must be allowed.

| Parameter | Result |
|-----------|--------|
| {...}      | Null  |

```
=> <id:42, method:"setOptions", path:"test/sh/a3">i{1:{"idleTimeOut":60}}
<= <id:42>i{}
```

## `*/ASSIGNED:close`

| Name    | SHV Path                   | Flags | Access        |
|---------|----------------------------|-------|---------------|
| `close` | Bellow Bytes Exchange node |       | Super-service |

This method allows caller to terminate the connection.

The access level applies only to the callers that do not match CallerIds.
Request messages with CallerIds matching that for `*:newExchange` that created
this node must be allowed.

| Parameter | Result |
|-----------|--------|
| Null      | Null   |

```
=> <id:42, method:"close", path:"test/sh/a3">i{}
<= <signal:"lsmod", path:"test/sh", source:"ls">i{1:{"a3":false}}
<= <id:42>i{}
```

## `*/ASSIGNED:peer`

| Name   | SHV Path                   | Flags  | Access        |
|--------|----------------------------|--------|---------------|
| `peer` | Bellow Bytes Exchange node | Getter | Super-service |

This provide ClientIds from `*:newExchange` that created this sub-node.

This method contrary to others of this node uses normal access level control and
thus only clients with Super-service access level can call it.

| Parameter | Result     |
|-----------|------------|
| Null      | [Int, ...] |

The result is List of Ints.

```
=> <id:42, method:"peer", path:"test/sh/a3">i{}
<= <id:42>i{2:[30,1,2]]}
```


## Walkthrough of the caller

The first step is to initiate the new connection by calling `*:newExchange`.
This will provide you with sub-node name that you will be using from now on
(instead of `ASSIGNED`).

The next step is to subscribe for the notifications for this node (The
expectation is that there is a RPC Broker in between. You can skip this step if
that is not the case). You want to use full path to the assigned node, source
`exchange` and all of signal `ready`.

Based on the usage you might want to inspect and tweak exchange options with
`*/ASSIGNED:options` and `*/ASSIGNED:setOptions` respectively.

The initial exchange should not contain any data and is intended only to query
for exchange state, but it is allowed to specify that it is ready to receive
non-zero amount of bytes. This allows answerer to send data even on initial
exchange.

The subsequent exchanges must contain only number of bytes the latest exchange
or `ready` signal specified as being free (the latest information applies).

It is highly suggested to prioritize data retrieval because it is possible that
more data can't be received by answerer unless you take some data out and thus
you might wait for non-zero ReadyToReceive indefinitely.

You also must maintain exchanges (even dummy ones that do not transfer any data)
in reasonable intervals to ensure that connection is not closed due to
inactivity (suggestion is the half interval of `idleTimeOut`).

The connection can be terminated by calling `*/ASSIGNED:close`. The answerer can
terminate connection for what ever reason on its own. It will send `*:ls:lsmod`
signal with but in general caller detects termination by receiving
`MethodNotFound` error when calling `*/ASSIGNED:exchange` method. There is no
way to report disconnect reason.


## Walkthrough of the answerer

The connection gets initiated by caller with `*:newExchange` method. Answerer
must allocate sub-node for it. Based on the intended usage the resource needed
for bytes exchanging can be initiated (such as spawning process or opening the
serial device) or it can be postponed to the first bytes exchange.

The data received on exchange will be processed (commonly just pushed to buffer)
and response will be created with data from processing. The sent data must be
kept until next exchange alongside with received Counter value to send same data
if Counter of the next exchange is the same. The received data from exchange
that has matching Counter with previous one are disregarded and not processed
(they are repeat of what was already received).

The number of bytes ready that can be received is provided to the caller as well
as number of bytes that could be still provided. The signal `ready` must be
emitted if one of these parameters changes from zero to non-zero because caller
might not attempt next exchange for some time if they are set to zero.

The idle time between exchanges is measured and if there was no exchange for
longer than `idleTimeOut` seconds then node is discarded and thus subsequent
calls for data exchange will result into an error `MethodNotFound`. The same
happens if `*/ASSIGNED:close` method is called.


## Design remarks

These are few points you should be aware when you are using Bytes Exchange
nodes:

* The access to the node is controlled by path of message in SHV network. This
  path is in general constant during the connection, but there are cases when it
  can change without client being notified about it (when intermediate RPC
  Broker resets) and this will result into two issues:

  * Existing connections will be inaccessible (resulting into `MethodNotFound`
    errors)
  * Existing connection could be suddenly accessible by some other client. The
    exploit of this would require pretty significant setup but still is
    possible.

* The reason for every connection being its own sub-node instead of method
  parameter is to allow filtering signals for a different connections and
  callers.

* The usage for `ready` signal is to signal only readiness when exchange wasn't
  previously ready. It should not intentionally be emitted if previous exchange
  specified no ability to receive or no ability to send bytes. This is to only
  fasten caller's detection of readiness when it is reasonable to just be quiet
  and keep answerer to work.

* The possibility of lost messages is covered by counter. This provides a way to
  easily get data even when messages are being randomly lost by attempting call
  again and again with same counter value. In case of caller these multiple call
  attempts are intuitive (they must contain same content) but answerer must
  ensure the following to keep data consistent:

  * Sent data are kept until next exchange to allow their re-transmit if counter
    isn't changed.
  * Received data are used only when counter changes, not when it stays the
    same.
