# File nodes

File node provides read and optionally also write access for the binary files
over SHV RPC.

The file directories are just regular nodes because the only required
functionality (that is listing files) is already provided by nodes discovery.

File nodes should not have any child nodes (`*:ls` should always return `[]`).

The access levels for these methods are only suggestions (compared to some other
method definition in this standard). That is because sometimes you want to allow
a different levels of access to different users to same file (such as append to
regular user, while write and truncate to super users).

The combination of provided method for file node will define the expected
functionality of the file. For the convinience of reader the following table is
provided with various, but not all, types of file node modes. The methods
`*:stat`, and `*:size` are always provided.

|             | `*:crc` / `*:sha1` | `*:read` | `*:write` | `*:truncate` | `*:append` |
|-------------|--------------------|----------|-----------|--------------|------------|
| Read only   | ✔️                  | ✔️        | ❌        | ❌           | ❌         |
| Fixed size  | ✔️                  | ✔️        | ✔️         | ❌           | ❌         |
| Resizable   | ✔️                  | ✔️        | ✔️         | ✔️            | ✔️          |
| Append only | ✔️                  | ✔️        | ❌        | ❌           | ✔️          |

## `*:stat`

| Name   | SHV Path | Flags  | Param Type | Result Type | Access |
|--------|----------|--------|------------|-------------|--------|
| `stat` | Any      | Getter |            | `!stat`     | Read   |

This method provides information about this file. It is required for file nodes.

The result is IMap with these fields:

| Key | Name       | Type             | Description                                                                                               |
|-----|------------|------------------|-----------------------------------------------------------------------------------------------------------|
| 0   | Type       | Int              | Type of the file (at the moment only regular is supported and thus must always be `0`)                    |
| 1   | Size       | Int              | Size of the file in bytes                                                                                 |
| 2   | PageSize   | Int              | Page size (ideal size and thus alignment for this file efficient access)                                  |
| 3   | AccessTime | DateTime \| Null | Optional time of latest data access                                                                       |
| 4   | ModTime    | DateTime \| Null | Optional time of latest data modification                                                                 |
| 5   | MaxWrite   | Int \| Null      | Optional maximal size in bytes of a single write that is accepted (this affects `*:write` and `*:append`) |

```
=> <id:42, method:"stat", path:"test/file">i{}
<= <id:42>i{2:i{0:1,1:3674,2:1024}}
```

## `*:size`

| Name   | SHV Path | Flags  | Param Type | Result Type | Access |
|--------|----------|--------|------------|-------------|--------|
| `size` | Any      | Getter |            | `i(0,)`     | Read   |

This provides faster access to only file size. Although it is also part of the
`*:stat` the size of the file is commonly used to quickly identify added data to
the file and thus it is provided separately as well. This method must be
implemented together with `*:stat`.

```
=> <id:42, method:"size", path:"test/file">i{}
<= <id:42>i{2:3674}
```

## `*:crc`

| Name  | SHV Path | Flags | Param Type                        | Result Type | Access |
|-------|----------|-------|-----------------------------------|-------------|--------|
| `crc` | Any      |       | `[i(0,):offset,i(0,)\|n:size]\|n` | `u(>32)`    | Read   |

The validation of the data is common operation that can be done either to verify
that all went all right or to detect that write might not be required because
data is already present. To prevent from doing this validation by pulling the
whole content of the file this CRC32 calculating method is provided. The client
providing the file node will calculate the CRC32 instead and send only that.

This method must be implemented alongside with `*:read` but it can also be
implemented on its own if you allow validation but not read for some reason
(make sure that in such case client can't read the file anyway with short
crc calculation ranges).

CRC32 algorithm to be used is the one used in IEEE 802.3 and [known as plain
CRC-32](https://reveng.sourceforge.io/crc-catalogue/all.htm#crc.cat.crc-32-iso-hdlc).

You can either pass argument `Null` and in such case checksum of the whole file
will be provided, or you can pass list with offset and size (that can be `Null`
for the all data up to the end of the file) in bytes that identifies range CRC32
should be calculate for.

No error must be raised if range specified by parameter is outside of the file.
Only bytes present in the range are used to calculate CRC32 (this includes case
when there are no bytes captured by specified range). This allows easier work
with files that are growing.

```
=> <id:42, method:"crc", path:"test/file">i{}
<= <id:42>i{2:409417892}
```
```
=> <id:42, method:"crc", path:"test/file">i{1:[1024, 2048]}
<= <id:42>i{2:25819716}
```

## `*:sha1`

| Name   | SHV Path | Flags | Param Type                        | Result Type | Access |
|--------|----------|-------|-----------------------------------|-------------|--------|
| `sha1` | Any      |       | `[i(0,):offset,i(0,)\|n:size]\|n` | `b(20)`     | Read   |

The basic CRC32 algorithm, that is required alongside with `*:read`, is
serviceable but it has higher probability of result collision. If you want to
use it to not only detect modification of the file but instead identification of
it, then SHA1 is the better choice. The implementation of this method is
optional (but when it exists the `*:crc` must exist as well).

You can either pass `Null` and in such case sha1 of the whole file will be
provided, or you can pass list with offset and size (that can be `Null` for the
all data up to the end of the file) in bytes that identifies range the hash
should be calculate for.

No error must be raised if range specified by parameter is outside of the file.
Only bytes present in the range are used to calculate SHA1 (this includes case
when there are no bytes captured by specified range). This allows easier work
with files that are growing.

The result must always be 20 bytes long *Bytes*.

```
=> <id:42, method:"sha1", path:"test/file">i{}
<= <id:42>i{2:b"4\972t\cc\efj\b4\df\aa\f8e\99y/\a9\c3\feF\89"}
```

## `*:read`

| Name   | SHV Path | Flags             | Param Type                  | Result Type | Access |
|--------|----------|-------------------|-----------------------------|-------------|--------|
| `read` | Any      | LARGE_RESULT_HINT | `[i(0,):offset,i(0,):size]` | `b`         | Read   |

Method for reading data from file. This method should be implemented only if you
allow reading of the file.

The parameter must be a list containing *offset* and *size* in bytes that
identifies the range to be read. The implementation may return less data than
*size*, but it will never return 0 bytes if any data exists at the specified
offset. The range can be outside of the file boundaries and in such case zero
length blob is returned.

```
=> <id:42, method:"read", path:"test/file">i{1:[0, 1024]}
<= <id:42>i{2:b"Hello World!"}
```
```
=> <id:42, method:"read", path:"test/file">i{1:[1024, 1024]}
<= <id:42>i{2:b""}
```

## `*:write`

| Name    | SHV Path | Flags | Param Type              | Result Type | Access |
|---------|----------|-------|-------------------------|-------------|--------|
| `write` | Any      |       | `[i(0,):offset,b:data]` |             | Write  |

Write is optional method that can be provided if modification of the file over
SHV RPC is allowed.

The parameter must be list with offset in bytes and bytes to be written on this
offset address.

Write outside of the current file boundary is up to the implementation. It can
extend file boundary if that is possible, or it can result into an error.

```
=> <id:42, method:"write", path:"test/file">i{1:[0, b"Hello World!"]}
<= <id:42>i{}
```

## `*:truncate`

| Name       | SHV Path | Flags | Param Type | Result Type | Access |
|------------|----------|-------|------------|-------------|--------|
| `truncate` | Any      |       | `i(0,)`    |             | Write  |

Change the file boundary. This method should be implemented if `*:write` allows
write outside of the file boundary. It should not be present if that is not
possible. In other words: presence of this method signals that write outside
file's boundary is possible alongside with presence of `*:write`.

The parameter must be requested size of the file.

It is up to the implementation if some boundaries to file change are imposed,
such as only increase is allowed, or maximal size of the file.

```
=> <id:42, method:"truncate", path:"test/file">i{1:1024}
<= <id:42>i{}
```

## `*:append`

| Name     | SHV Path | Flags | Param Type | Result Type | Access |
|----------|----------|-------|------------|-------------|--------|
| `append` | Any      |       | `b`        |             | Write  |

Append is a optional special way to perform write by always appending to the end
of the file. Append can be provided even if `*:write` is not and other way
around.  `*:truncate` also doesn't have to be implemented, append is always
outside file boundary.

The parameter is sequence of bytes to be appended to the file.

```
=> <id:42, method:"truncate", path:"test/file">i{1:b"Some text..."}
<= <id:42>i{}
```
