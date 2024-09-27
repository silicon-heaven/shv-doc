# Device API

Device is a special application that represents a single physical device. It is
benefical to see the difference between random application and application that
runs in dedicated device and controls such device. This allows generic
identification of such devices in the SHV tree.

The call to `:ls(".app")` can be used to identify application as being a
device.

## `.device:name`

| Name   | SHV Path      | Flags  | Access |
|--------|---------------|--------|--------|
| `name` | `.device`     | Getter | Browse |

This method must provide the device name. This is a specific generic name of the
device.

| Parameter | Result |
|-----------|--------|
| Null      | String |

```
=> <id:42, method:"name", path:".device">i{}
<= <id:42>i{2:"OurDevice"}
```

## `.device:version`

| Name      | SHV Path      | Flags  | Access |
|-----------|---------------|--------|--------|
| `version` | `.device`     | Getter | Browse |

This method must provide version (revision) of the device.

| Parameter | Result |
|-----------|--------|
| Null      | String |

```
=> <id:42, method:"name", path:".device">i{}
<= <id:42>i{2:"g2"}
```

## `.device:serialNumber`

| Name           | SHV Path      | Flags  | Access |
|----------------|---------------|--------|--------|
| `serialNumber` | `.device`     | Getter | Browse |

This method can provide serial number of the device if that is something the
device has. It is allowed to provide *Null* in case there is no serial number
assigned to this device.

| Parameter | Result         |
|-----------|----------------|
| Null      | String \| Null |

```
=> <id:42, method:"serialNumber", path:".device">i{}
<= <id:42>i{2:"12590"}
```

## `.device:uptime`

| Name     | SHV Path  | Flags  | Access |
|----------|-----------|--------|--------|
| `uptime` | `.device` | Getter | Read   |

This provide current device's uptime in seconds. It is allowed to provide *Null*
in case device doesn't track its uptime.

| Parameter | Result       |
|-----------|--------------|
| Null      | UInt \| Null |

```
=> <id:42, method:"uptime", path:".device">i{}
<= <id:42>i{2:3842}
```

## `.device:reset`

| Name    | SHV Path  | Flags | Access  |
|---------|-----------|-------|---------|
| `reset` | `.device` |       | Command |

Initiate the device's reset. This might not be implemented and in such case
`NotImplemented` error should be provided.

| Parameter | Result       |
|-----------|--------------|
| Null      |  Null |

```
=> <id:42, method:"reset", path:".device">i{}
<= <id:42>i{}
```

## `.device/alerts:get`

| Name  | SHV Path         | Flags  | Access |
|-------|------------------|--------|--------|
| `get` | `.device/alerts` | Getter | Read   |

Get the current device's alerts.

| Parameter   | Result         |
|-------------|----------------|
| Null \| Int | \[i{...},...\] |

* `0` (*date*): date and time of the alert creation.
* `1` (*level*): int with notice level. This is value between 0 and 63 that can
  be used to sort alerts. The classification is to the three named levels:
  * Notice: from 0 to 20. These are alerts signal that do not affect nor primary
    nor secondary device's functionality but require some attention. These can
    also be potential issues that would affect the device's secondary
    functionality.
  * Warning: from 21 to 42. These are levels to be used for issues not affecting
    the device's primary functionality or notices for possible issues that would
    affect the primary functionality.
  * Error: from 43 to 63. These should be used for issues affecting the primary
    functionality of the device.
* `2` (*id*): string with notice identifier. This identifier should be chosen to
  be unique for not for the single device but also between devices. It should be
  short but at the same time precise and unique. The benefit is human
  readability. This ID then should be used in device's manual.
* `3` (*info*): any SHV value that provides additional info. This is optional
  field that can be used to pass some additional info related to the alert. The
  value interpretation should be documented but is outside of the SHV scope.

```
=> <id:42, method:"get", path:".device/alerts">i{}
<= <id:42>i{2:[i{0:d"2017-05-03T15:52:31.123", 1:42, 2:"EX_CMP_TEST"}]}
```

## `.device/alerts:get:chng`

The alerts change must be signaled with `chng` signal.

| Value          |
|----------------|
| \[i{...},...\] |

```
<= <signal:"chng", path:".device/alerts", source:"get">i{1:[]}
```
