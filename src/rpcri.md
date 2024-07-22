# SHV RPC RI

This is definition of Resource Identifiers for SHV RPC. SHV has two types of
resources; those are methods and signals. Signals are always associated with
some method and thus Resource Identifier actually matches primarily methods
while it can also select between different signals.

Resource identifiers are human readable and are used thorough this
documentation, in the broker's signal filtering. They can also be used by other
tools.

Example of Resource Identifiers for SHV RPC:
```
.app:name
test/device/track/1:*:chng
test/*:ls:lschng
```

Resource Identifier consists of up to triplet delimited by colon `:`. The
following variations are allowed:

```
PATH
PATH:METHOD
PATH:METHOD:SIGNAL
PATH:METHOD:
```

__The first field `PATH` is the SHV path.__ It can be glob pattern (rules from
POSIX.2, 3.13 with added support for ``**`` that matches multiple nodes). Note
that empty `PATH` is the root of the SHV node tree.

__The second field `METHOD` is the SHV RPC method name.__ It can be wildcard
pattern (rules from POSIX.2, 3.13). The method name can't be empty. It is
assumed to be `*` if not specified at all (the `PATH` variation).

__The third field `SIGNAL` is the SHV RPC signal name__ while `METHOD` in such
case is its source. It can be wildcard pattern (rules from POSIX.2, 3.13). It is
assumed to be `*` if not specified (the `PATH` and `PATH:METHOD` variant). The
empty signal name is invalid and thus it is used to identify only method (thus
`PATH:METHOD:` matches only method while `PATH:METHOD` matches also all of its
signals).

The following table breaks down different variations and how they are applied
when matching either method or signal:

| Variation            | Method                                        | Signal                                                      |
|----------------------|-----------------------------------------------|-------------------------------------------------------------|
| `PATH`               | Applies if `PATH` matches.                    | Applies if `PATH` matches.                                  |
| `PATH:METHOD`        | Applies if `PATH` and `METHOD` matches.       | Applies if `PATH` and `METHOD` matches.                     |
| `PATH:METHOD:SIGNAL` | Applies if `PATH` and `METHOD` matches.       | Applies if `PATH` and `METHOD` and `SIGNAL` matches.        |
| `PATH:METHOD:`       | Applies if `PATH` and `METHOD` matches.       | Never applies                                               |

Notice that there is no variation that matches only signals and not its
associated method. This is because signal is always associated with some method
and Resource Identifier primarily identifies the method and only after that
filters its signals if resource is signal. This means that Resource Identifier
can't be used to filter both methods and signals from the same set of resources.

The examples of different Resource Identifiers and resources:

| Resource                                                         | `**` | `**:get:` | `test/**` | `test/**:get:*chng` |
|------------------------------------------------------------------|------|-----------|-----------|---------------------|
| Method `name` on path `.app`                                     | ✔️    | ❌        | ❌        | ❌                  |
| Method `get` on path `sub/device/track`                          | ✔️    | ✔️         | ❌        | ❌                  |
| Method `get` on path `test/device/track`                         | ✔️    | ✔️         | ✔️         | ✔️                   |
| Signal `chng` associated with `get` on path `test/device/track`  | ✔️    | ❌        | ✔️         | ✔️                   |
| Signal `lschng` associated with `ls` on path `test/device/track` | ✔️    | ❌        | ✔️         | ❌                  |
