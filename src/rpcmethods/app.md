# Application API

These are methods that are required for every device to be present on its SHV
path `".app"`. Clients do not have to implement these, but their implementation
is highly suggested if they are supposed to be connected to the broker for more
than just a few requests.

## `.app:shvVersionMajor`

| Name              | SHV Path | Flags  | Param Type | Result Type | Access |
|-------------------|----------|--------|------------|-------------|--------|
| `shvVersionMajor` | `.app`   | Getter |            | `i`         | Browse |

This method provides information about implemented SHV standard. Major version
number signal major changes in the standard and thus you are most likely
interested just in this number.

```
=> <id:42, method:"shvVersionMajor", path:".app">i{}
<= <id:42>i{2:0}
```

## `.app:shvVersionMinor`

| Name              | SHV Path | Flags  | Param Type | Result Type | Access |
|-------------------|----------|--------|------------|-------------|--------|
| `shvVersionMinor` | `.app`   | Getter |            | `i`         | Browse |

This method provides information about implemented SHV standard. Minor version
number signals new features added and thus if you wish to check for support of
these additions you can use this number.

```
=> <id:42, method:"shvVersionMinor", path:".app">i{}
<= <id:42>i{2:1}
```

## `.app:name`

| Name   | SHV Path | Flags  | Param Type | Result Type | Access |
|--------|----------|--------|------------|-------------|--------|
| `name` | `.app`   | Getter |            | `s`         | Browse |

This method must provide the name of the application, or at least the SHV
implementation used in the application.

```
=> <id:42, method:"name", path:".app">i{}
<= <id:42>i{2:"SomeApp"}
```

## `.app:version`

| Name      | SHV Path | Flags  | Param Type | Result Type | Access |
|-----------|----------|--------|------------|-------------|--------|
| `version` | `.app`   | Getter |            | `s`         | Browse |

This method must provide the application version, or at least the SHV
implementation used in the application (must be consistent with information in
`.app:appName`).

```
=> <id:42, method:"version", path:".app">i{}
<= <id:42>i{2:"1.4.2-s5vehx"}
```

## `.app:ping`

| Name   | SHV Path | Flags | Param Type | Result Type | Access |
|--------|----------|-------|------------|-------------|--------|
| `ping` | `.app`   |       |            |             | Browse |

This method should reliably do nothing and should always be successful. It is
used to check the connection (if message can be passed to and from client) as
well as to keep connection in case of SHV Broker.

```
=> <id:42, method:"ping", path:".app">i{}
<= <id:42>i{}
```

## `.app:date`

| Name   | SHV Path | Flags | Param Type | Result Type | Access |
|--------|----------|-------|------------|-------------|--------|
| `date` | `.app`   |       |            | `t`         | Browse |

This is an optional method that provides access to the date and time this
application is using (that includes time zone). Applications running on systems
without RTC are not expected to implement this method. You must implement this
any time methods this application provides to SHV works with date and time.

You should use this to detect time shift between your time and time of the
device you are talking to. Date and time sent by device will be relative to this
one and thus even if it has wrong time set you have change to calculate the
correct one. The same applies the other way around, but in general such methods
should be avoided.

Note that there is unspecified overhead of SHV RPC network in up to seconds for
transferring messages and thus precision of comparison with local time must
consider this.

```
=> <id:42, method:"date", path:".app">i{}
<= <id:42>i{2:d"2017-05-03T15:52:31.123"}
```
