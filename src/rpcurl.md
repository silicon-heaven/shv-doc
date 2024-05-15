# SHV RPC URL

Unified Resource Locators are common way to specify connection. This is
definition of such URL for Silicon Heaven RPC.

Examples of URLs for Silicon Heaven RPC:

```
tcp://user@localhost:3755?password=pass
tcp://user@localhost:3755?password=pass&devid=42
unix:/run/shvbroker.sock
```

The base format is:

```
URL = scheme ":" ["//" [username "@"] authority] [path] ["?" options]
```

`options` is sequence of attribute-value pairs split by `=` an joined by
`&` (example: `password=test&devid=foo`). Generally supported options are:

* `password`: Plain text password used to login to the server.
* `shapass`: Password hashed with SHA1 used to login to the server.
* `user`: Alternative way to set user name that overrides user in URL. The
  default user name if neither is used is local user's name or the platform
  specific alternative.
* `devid`: Identify to the other side as device with this ID.
* `devmount`: Identify to the other side as device and request mount to
  the given location.


## TCP/IP protocol

```
scheme = "tcp"
authority = host [":" port]
```

The default `host` is `localhost` and `port` is `3755`. Any non-empty `path` is
invalid as it has no meaning in IP.

This uses TCP/IP with [Block transport layer](rpctransportlayer.md#stream).


## TCP/IP serial protocol

```
scheme = "tcps"
authority = host [":" port]
```

This is variant of TCP/IP protocol. It is same except of used transport layer.
Please refer to the previous section for more info. The only difference is the
default `port` that is `3765`.

This uses TCP/IP with [Serial transport layer](rpctransportlayer.md#serial).


## TCP/IP protocol with SSL

```
scheme = "ssl"
authority = host [":" port]
```

The default `host` is `localhost` and `port` is `3756`. Any non-empty `path`
is invalid as it has no meaning in IP.

The additional supported options are:

* `ca`: Path to the file with CA certificates used to verify the peer.
* `cert`: Path to the file with certificate. For clients connecting to the
  server this is client certificate that is validated by server for access and
  in some cases can replace password. For server this is certificate clients
  verify to validate if they are connecting to the correct server.
* `key`: Path to the file with secret part of the `cert`. This must be specified
  alongside with `cert`.
* `crl`: Path to the file with certification revocation list. This is used to
  invalidate client certificates on the server.
* `verify`: can be used with either `true` or `false` to control if server
  should be verified or not. The default, if not specifies, is `true`. Setting
  `false` forces client to accept any certificate as valid.

This uses TLS TCP/IP with [Block transport layer](rpctransportlayer.md#stream).


## TCP/IP serial protocol with SSL

```
scheme = "ssls"
authority = host [":" port]
```

This is variant of TCP/IP protocol with SSL. It is same except of used transport
layer. Please refer to the previous section for more info. The only difference
is the default `port` that is `3766`.

This uses TLS TCP/IP with [Serial transport layer](rpctransportlayer.md#serial).


## Unix/Local domain socket

```
scheme = "unix"
```

There is no default path and thus empty `path` is considered invalid. Any
non-empty `authority` is also considered as invalid because it has no meaning.

This uses Unix sockets for local interprocess communication with [Block
transport layer](rpctransportlayer.md#stream).


## Unix/Local domain socket serial protocol

```
scheme = "unixs"
```

This is variant of Unix/Local domain socket. It is same except of used transport
layer. Please refer to the previous section for more info.

This uses Unix sockets for local interprocess communication with [Serial
transport layer](rpctransportlayer.md#serial).


## Serial / RS232

```
scheme = ("serial" | "tty")
```

`path` needs to point to valit serial device. There is no default path and thus
empty `path` is considered invalid. Any non-empty `authority` is also considered
as invalid because it has no meaning.

The additional supported options are:

* `baudrate`: Specifies baudrate used for the serial communication.

Other common serial-port parameters are at the moment specified as not
configurable and are expected to be: eight bits per word, no parity, single stop
bit, enabled hardware flow control, disabled software flow control.

This uses serial console or terminal like interface as bidirectional stream
channel with [Serial transport layer](rpctransportlayer.md#serial).

## WebSocket

```
scheme = ("ws")
authority = host [":" port]
```

The default `host` is `localhost` and `port` is `8755`.

## WebSocket over SSL

```
scheme = ("wss")
authority = host [":" port]
```

The default `host` is `localhost` and `port` is `8766`.

* `ca`: Path to the file with CA certificates used to verify the peer.
* `cert`: Path to the file with certificate. For clients connecting to the
  server this is client certificate that is validated by server for access and
  in some cases can replace password. For server this is certificate clients
  verify to validate if they are connecting to the correct server.
* `key`: Path to the file with secret part of the `cert`. This must be specified
  alongside with `cert`.
* `crl`: Path to the file with certification revocation list. This is used to
  invalidate client certificates on the server.
* `verify`: can be used with either `true` or `false` to control if server
  should be verified or not. The default, if not specifies, is `true`. Setting
  `false` forces client to accept any certificate as valid.
