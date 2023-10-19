# Application API

These are methods that are required for every device to be present on its SHV
path `".app"`. Clients do not have to implement these, but their implementation
is highly suggested if they are supposed to be connected to the broker for more
than just a few requests.

## `.app:shvVersionMajor`

| Name              | SHV Path | Signature   | Flags  | Access |
|-------------------|----------|-------------|--------|--------|
| `shvVersionMajor` | `.app`   | `ret(void)` | Getter | Browse |

This method provides information about implemented SHV standard. Major version
number signal major changes in the standard and thus you are most likely
interested just in this number.

| Parameter | Result |
|-----------|--------|
| Null      | Int    |

```
=> <id:42, method:"shvVersionMajor", path:".app">i{}
<= <id:42>i{2:0}
```

## `.app:shvVersionMinor`

| Name              | SHV Path | Signature   | Flags  | Access |
|-------------------|----------|-------------|--------|--------|
| `shvVersionMinor` | `.app`   | `ret(void)` | Getter | Browse |

This method provides information about implemented SHV standard. Minor version
number signals new features added and thus if you wish to check for support of
these additions you can use this number.

| Parameter | Result |
|-----------|--------|
| Null      | Int    |

```
=> <id:42, method:"shvVersionMinor", path:".app">i{}
<= <id:42>i{2:1}
```

## `.app:name`

| Name      | SHV Path | Signature   | Flags  | Access |
|-----------|----------|-------------|--------|--------|
| `name` | `.app`   | `ret(void)` | Getter | Browse |

This method must provide the name of the application, or at least the SHV
implementation used in the application.

| Parameter | Result |
|-----------|--------|
| Null      | String |

```
=> <id:42, method:"name", path:".app">i{}
<= <id:42>i{2:"SomeApp"}
```

## `.app:version`

| Name         | SHV Path | Signature   | Flags  | Access |
|--------------|----------|-------------|--------|--------|
| `version` | `.app`   | `ret(void)` | Getter | Browse |

This method must provide the application version, or at least the SHV
implementation used in the application (must be consistent with information in
`.app:appName`).

```
=> <id:42, method:"version", path:".app">i{}
<= <id:42>i{2:"1.4.2-s5vehx"}
```

## `.app:ping`

| Name   | SHV Path | Signature    | Flags | Access |
|--------|----------|--------------|-------|--------|
| `ping` | `.app`   | `void(void)` |       | Browse |

This method should reliably do nothing and should always be successful. It is
used to check the connection (if message can be passed to and from client) as
well as to keep connection in case of SHV Broker.

| Parameter | Result |
|-----------|--------|
| Null      | Null   |

```
=> <id:42, method:"ping", path:".app">i{}
<= <id:42>i{}
```
