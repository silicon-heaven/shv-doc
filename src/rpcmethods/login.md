# Login sequence

The login sequence needs to be performed every time a new client is connected to
the broker. It ensures that client gets correct access rights assigned and that
devices get to be mounted in the correct location in the broker's nodes tree.
Sometimes the authentication can be optional (such as when connecting over local
socket or if client certificate is used) but login sequence is required anyway
to introduce the client to the broker.

The first method to be sent by the client after connection to the broker is
established needs to be `hello` request. It is called with `null` SHV path
(which means it can be left out in the message's meta) and no parameters.

```
=> <id:1, method:"hello">i{}
```

Broker should respond with message containing nonce that can be used to perform
login. The nonce needs to be an ASCII string with length from 10 to 32
characters. The nonce must be the same in case, of `hello` message retransmit 
during the login phase. Some clients might send more `hello` message to discover,
that shvbroker is started and ready, especially on serial port. 

```
<= <id:1>i{2:{"nonce":"vOLJaIZOVevrDdDq"}}
```

The next message sent by the client needs to be `login` request. This message is
*Map* and needs to contain `"login"` if login is required and can contain
`"options"`.

There are two types of the logins you can use. It can be either *plain* login or
*sha1*. The difference is only the way you send the password in the message. The
*plain* login just sends password as it is in *String*. The *sha1* login hashes
the provided user's password, adds the result as suffix to the nonce from `hello`
and hashes the result again. SHA1 hash is represented in HEX format. The
complete password deduction is: `SHA1(nonce + SHA1(password))`. The SHA1 login
is highly encouraged and plain login is highly discouraged, even the low end
CPUs should be able to calculate SHA1 hash and thus perform the SHA1 login. The
`"login"` *map* needs to contain these fields:

* `"user"` with username as *String*
* `"password"` with either plain text password or SHA1 password (as described in
  the paragraph above) as *String*
* `"type"` that identifies password format, and thus it can have only one of
  these values:
  * `"PLAIN"` for *plain* text password (discouraged use)
  * `"SHA1"` for SHA1 login

The `"options"` needs to be a map of options for the broker. Broker will ignore
any unknown options. A different broker implementations can support a different
set of options, but minimal needed supported options are these:

* `"device"` with *map* containing device specific options. At least these
  options need to be supported:
  * `"deviceId"` with *String* value that identifies the device. This should be
    used by the broker to automatically assign mount point based on its internal
    rules.
  * `"mountPoint"` with *String* value that specifies path where the device's
    tree should be mounted in the broker. This should overwrite any mount point
    assigned by broker based on `"deviceId"`, but it can be disregarded if user
    doesn't have access rights for the SHV path specified.
* `"idleWatchDogTimeOut"` specifies number of seconds without message receive
  before broker will consider the connection to be dead and disconnects it. The
  default timeout by the broker should be 180 seconds. By increasing this
  timeout you can reduce the periodic dummy messages sent, but it is suggested to
  keep it in reasonable range because open but dead connection can consume
  unnecessary resources on the broker.

```
=> <id:2, method:"login">i{1:{"login":{"password":"3d613ce0c3b59a36811e4acbad533ee771afa9f3","user":"iot","type":"SHA1"}}, "options":{"device":{"deviceId":"bfsview_test", "mountPoint":"test/bfsview"}, "idleWatchDogTimeOut":180}}}
```

The broker will respond with *Null* (some older broker implementation can
respond with some value for backward compatibility reasons). From now on you can
send any requests and receive any messages you have rights on as logged user
from the broker.

```
<= <id:2>i{}
```

In case of a login error you can attempt the login again without need to
disconnect or sending `hello` again. Be aware that broker should impose delay of
60 seconds on subsequent login attempts for security reasons.

> Note that `hello` and `login` methods are not to be reported by `dir` and no
> other method can be called before login sequence is completed.
