# SHV Journal [DRAFT]

## `.app/shvjournal`

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
