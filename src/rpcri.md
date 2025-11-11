# SHV RPC RI

This is definition of Resource Identifiers for SHV RPC. SHV has two types of
resources; those are methods and signals.

Resource identifiers are human readable and are used thorough this
documentation as well as in the broker's signal filtering. They can also be used
by other tools.

The format is either `PATH:METHOD` for methods or `PATH:METHOD:SIGNAL` for
signals.

__The first field `PATH` is the SHV path.__ It can be glob pattern (rules from
POSIX.2, 3.13 with added support for ``**`` that matches multiple nodes). In
contrast to the Bash globstar implementation, the SHV glob-star pattern `**`
matches zero or more directories at the end of the path. For example `foo/**`
matches `foo`, `foo/bar`, `foo/bar/baz`, etc. By comparison, Bash’s `foo/**`
pattern matches `foo/bar` and `foo/bar/baz`, but not `foo` itself. Note that
empty `PATH` is the root of the SHV node tree.

__The second field `METHOD` is the SHV RPC method name.__ It can be wildcard
pattern (rules from POSIX.2, 3.13). The empty method name is invalid and thus is
invalid pattern.

__The third field `SIGNAL` that is used only for signals is the SHV RPC signal
name__ while `METHOD` in such case is its source. It can be wildcard pattern
(rules from POSIX.2, 3.13). The empty signal name is invalid and thus is invalid
pattern.

The examples of Resource Identifiers for methods and method matching:

| Resource                         | `**:*` | `**:get`  | `test/**:get` | `**:*:*` |
|----------------------------------|--------|-----------|---------------|----------|
| Method `.app:name`               | ✔️     | ❌        | ❌            | ❌       |
| Method `sub/device/track:get`    | ✔️     | ✔️        | ❌            | ❌       |
| Method `test:get`                | ✔️     | ✔️        | ✔️            | ❌       |
| Method `test/device/track:get`   | ✔️     | ✔️        | ✔️            | ❌       |

The examples of Resource Identifiers for signals and signals matching:

| Resource                             | `**:*:*` | `**:get:*` | `test/**:get:*chng` | `test/*:ls:lsmod` |
|--------------------------------------|----------|------------|---------------------|-------------------|
| Signal `test:get:chng`               | ✔️       | ✔️         | ✔️                  | ❌                |
| Signal `test/device/track:get:chng`  | ✔️       | ✔️         | ✔️                  | ❌                |
| Signal `test/device/track:get:mod`   | ✔️       | ✔️         | ❌                  | ❌                |
| Signal `test/device/track:ls:lsmod`  | ✔️       | ❌         | ❌                  | ✔️                |
