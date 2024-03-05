# Device API

Device is a special application that represents a single physical device. It is
benefical to see the difference between random application and application that
runs in dedicated device and controls such device. This allows generic
identification of such devices in the SHV tree.

The call to `.app:ls("device")` can be used to identify application as being a
device.

## `.device:name`

| Name   | SHV Path      | Signature   | Flags  | Access |
|--------|---------------|-------------|--------|--------|
| `name` | `.device`     | `ret(void)` | Getter | Browse |

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

| Name      | SHV Path      | Signature   | Flags  | Access |
|-----------|---------------|-------------|--------|--------|
| `version` | `.device`     | `ret(void)` | Getter | Browse |

This method must provide version (revision) of the device.

| Parameter | Result |
|-----------|--------|
| Null      | String |

```
=> <id:42, method:"name", path:".device">i{}
<= <id:42>i{2:"g2"}
```

## `.device:serialNumber`

| Name           | SHV Path      | Signature   | Flags  | Access |
|----------------|---------------|-------------|--------|--------|
| `serialNumber` | `.device`     | `ret(void)` | Getter | Browse |

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
