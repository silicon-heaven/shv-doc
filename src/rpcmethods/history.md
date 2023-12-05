# History [DRAFT]

## `.app/history`

Node where device history data is stored in log files. Every log file is named by timestamp of its beginning (i.e. `2023-09-11T09-00-00-000.log2`) to enable binary search in files to find correct log file.

### Log2 format

Simple text log, every record is on single line ended with `LF`. Record fields are separated by `TAB`.

Columns:
1. `MonotonicDateTime` - monotonic timestamp of record occurrence
2. `DeviceDateTime` - this field is filled only in rare case, when device time is not equal to monotonic one, for example after restart before NTP is started
3. `ShvPath`
4. `Value` - Cpon serialized value, note that Cpon cannot contain neither `TAB` nor `LF`.
5. `ShortTime` - very simple devices without precise time can use 16-bit cyclic short-time in msec. Using short-time we can measure time interval among log records, even if we do not have correct monotonic time.
6. `Domain` - kind of record. In almost all cases is domain equal do name of SHV signal emitted when record was created. `chng` in example below means 'value changed'.
7. `ValueFlags` - record value flags bitfield
   * bit 0 - `Snapshot` - value is originated in device snapshot, it is not spontaneous
   * bit 1 - `Spontaneous` - value change is spontaneous
   * bit 2 - `DirtyValue` - reserved by history provider, not used by any device
8. `UserId` - ID of user issuing shv method call, if method calls are logged by device

example
```
cat 2023-09-11T09-00-00-000.log2

2023-09-11T09:24:28.100Z		system/plcDisconnected	false		chng	1
2023-09-11T09:24:28.100Z		system/status	0u		chng	1
2023-09-11T09:24:28.100Z		system/name	"Ell038, Vystavka"		chng	1
2023-09-11T09:24:28.100Z		system/version	"V1.00 - 202-ell038 6bbdff5e - Sep 11 2023 11:17:16"		chng	1
2023-09-11T09:24:28.100Z		system/info/tempCPU	0		chng	1
2023-09-11T09:24:28.100Z		system/info/tempPLC	0		chng	1
2023-09-11T09:24:28.100Z		system/info/memoryUserSize	0		chng	1
2023-09-11T09:24:28.100Z		system/info/memoryUserFree	0		chng	1
2023-09-11T09:24:28.100Z		system/info/memoryDRAMFree	0		chng	1
2023-09-11T09:24:28.100Z		system/ethernet/MAC	""		chng	1
2023-09-11T09:24:28.100Z		system/ethernet/IP	""		chng	1
2023-09-11T09:24:28.100Z		system/ethernet/subnetMask	""		chng	1
```

### `.app/history:getSnapshot`

| Name        | SHV Path                    | Signature     | Flags | Access |
|-------------|-----------------------------|---------------|-------|--------|
| `getSnapshot` | `.app/history`            | `void()`      |       | Read   |

Returns device snapshot at current time.

```

<
  "dateTime":d"2023-12-05T13:54:45Z"
>{
  ".app/alarms/infoCount": 0,
  ".app/alarms/warningCount": 6,
  ".app/alarms/errorCount": 2,
  "system/plcDisconnected": false,
  "system/status": 1u,
  "system/name": "G3 Project",
  "system/version": "V1.00 - test 6a65d904 - Dec  4 2023 15:58:52",
  "system/deviceID": "000294D7000F",
  "system/info/tempCPU": 46,
  "system/info/tempPLC": 46,
  "system/info/memoryUserSize": 161,
  "system/info/memoryUserFree": 150,
  "system/info/memoryDRAMFree": 60,
  "system/ethernet/MAC": "00:60:65:5E:20:A6",
  "system/ethernet/IP": "10.0.0.34",
  "system/ethernet/subnetMask": "255.255.252.0",
  "system/ethernet/gatewayIP": "10.0.0.254",
  "system/ethernet/hostName": "SystemG3_plc",
  "system/datetime/NTPclientActive": true,
  "system/datetime/dateTime": d"2023-12-05T13:54:45Z",
  "system/PLCmodules/errorPLCModule": false,
  "system/PLCmodules/errorPLCModuleAddress": "",
  "system/superiorSystem/status": 1u,
  "system/superiorSystem/slaveMode": false,
  "devices/zone/Zone_V3/status": 258u,
  "devices/zone/Zone_V3/reasonAB": 1u,
  ...
}
```


### `.app/history:yyyy-mm-ddTHH-MM-SS-zzz.log2`

Every log file must provide functions to read it.

| Name   | SHV Path                                    | Return type | Param type | Flags  | Access |
|--------|---------------------------------------------|-------------|------------|--------|        |
| `size` | `.app/history:yyyy-mm-ddTHH-MM-SS-zzz.log2` | UInt        | void       |        | Read   |

Returns size of file in bytes.

| Name   | SHV Path                                    | Return type | Param type | Flags  | Access |
|--------|---------------------------------------------|-------------|------------|--------|        |
| `read` | `.app/history:yyyy-mm-ddTHH-MM-SS-zzz.log2`      | Blob | Map |       | Read   |

Returns `size` of file bytes starting on `offset` position.

| Param | Default value | Description |
|-------|--------|---------------|
| `offset` | 0 | Ofset where file is start to read   |
| `size` | MAX_INT | Maximum number of bytes returned by function `read()`   |
