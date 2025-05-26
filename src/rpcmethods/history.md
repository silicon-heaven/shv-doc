# History

History is functionality that allows logging of signals and access to the
history of changes of these signals. Client with history can provide its
history to upper implementations and that way histories can be propagated
through the SHV RPC network. This propagation allows central storage of the long
term history while peers lower in the network can aggregate only shorter ranges.
In other words: this API provides the following functionalities while clients
doesn't have to implement all of them:

* Aggregated access to the logs on time bases, that is method
  `.history/**:getLog`.
* Systematic retrieval of records (for other histories to use) by providing
  appropriate methods bellow `.history/**/.records` nodes.
* Systematic retrieval of file logs (for other histories to use) by providing
  appropriate nodes bellow  `.history/**/.files` nodes.

History can be implemented either directly as part of the application or it can
be mounted to SHV RPC Broker. It **must** always be mounted in `.history` mount
point, that is why all paths for this API are prefixed with `.history/`. Any
other mount point is not valid for RPC History because it should be at the same
node as `.app` is present.


## `.history/**`

History can only record signals and provide them for other histories (through
`.history/.records/*` and `.history/.files/*`) or it can also provide aggregated
access to logs it stores.

If history provides aggregated access to the logs then it must contain a virtual
tree of the logs sources. This allows setting limited log access based on the
same path as the real SHV RPC Broker node tree. This means that if history
records signal with SHV path `test/device/foo/status` then path
`.history/test/device/foo/status` must be valid and discoverable.

The aggregated access to the logs doesn't have to be provided for all the logs,
but all paths recorded (or to be recorded) in single log must be provided.

### `.history/**:getLog`

| Name     | SHV Path      | Flags           | Param Type | Result Type | Access |
|----------|---------------|-----------------|------------|-------------|--------|
| `getLog` | `.history/**` | HintLargeResult | `!getLogP` | `!getLogR`  | Browse |

Queries logs for the recorded signals in given time range.

This method must be available on all nodes in the `.history/**` tree with
exception of `.app`, `.records` and `.files`. In other words this must be
provided only if aggregated log access is provided and in such case it is
required.

The parameter is an *IMap* containing the following fields:

| Key | Name  | Type           |                        |
| --- | ----- | -------------- | ---------------------- |
| 1   | Since | DateTime\|Null | Defines the starting point for log retrieval. Records with a timestamp exactly matching this value are excluded, allowing seamless continuation from the last retrieved record. *Default:* The time at which the request is received. |
| 2   | Until | DateTime\|Null | Specifies the end point for log retrieval. Only records with a timestamp within the range `(Since, Until>` will be returned. If `Until` comes before `Since`, the interval becomes `<Until, Since)`. In such cases, logs records are returned in **reverse order** (from newest to oldest). If there are too many log records matching specified interval, the device may return fewer records and stop before reaching `Until` timestamp. To retrieve all logs, send a follow-up request using the timestamp of the last returned record as the new `Since`. *Default:* The time at which the request is received. |
| 3   | Count | Int\|Null      | Limits the maximum number of records returned. Device can provides number of records it decides on but sometimes it is desirable to lower this limit, for example to single record, and this option allows that. If multiple logs have the same timestamp, all must be included even if this exceeds the limit.  *Default:* Unlimited. The snapshot is returned always complete regardless `Count` value specified. |
| 4   | Ri    | [RPC Resource Identifier](../rpcri.md)\|Null | Filters the results to include only signals matching specified RI. Filter must be relative to `getLog` shv path. It is strongly recommended to call `getLog` using the most specific path possible instead of relying on the `Ri` field, as access permissions are determined by the request path. Using a shorter path may result in insufficient access rights. *Default:* Null, filter is off. |


The provided value is list of *IMap*s with following fields:

| Key | Name      | Type     |                     |
| --- | --------- | -------- | ------------------- |
| 1   | TimeStamp | DateTime | Timestamp of the record. |
| 2   | Ref       | Int      | It provides a way to reference the previous record to use it as the default for `Path`, `Signal` and `Source` (instead of the documented defaults). It is *Int* where `0` is record right before this one in the list. The offset must always be to the most closest record that specifies desired default. This simplifies parsing because there is no need to remember every single received record but only the latest unique ones. It is up to the implementation if this is used or not. Simple implementations can choose to generate bigger messages and not use this field at all. |
| 3   | Path      | String   | SHV path to the node relative to the path `getLog` was called on. *Default:* empty path `""`. |
| 4   | Signal    | String   | Signal name. *Default:* `chng`. |
| 5   | Source    | DateTime | Signal's associated method name.  *Default:* `get`. |
| 6   | Value     | RpcValue | Signal's value (parameter). |
| 7   | UserId    | String   | `UserId` carried by signal message. *Default:* `null`. |
| 8   | Repeat    | Bool     | `Repeat` carried by signal message. *Default:* `False`. |

  
The provided records should be sorted according to the `TimeStamp` field `1`
either in ascending order if `Since` < `Until` or descending order 
if `Until` < `Since`.

The method itself has only `Browse` access level but it must filter provided
logs based on their access level and thus user with low access level might not
see all that is provided. Note that it is not possible to decrease access level
of the user for some part of the SHV tree because he could always ask the upper
node where his access level is high enough and logs would be provided.

### `.history/**:getSnapshot`

| Name     | SHV Path      | Flags           | Param Type | Result Type | Access |
|----------|---------------|-----------------|------------|-------------|--------|
| `getSnapshot` | `.history/**` | HintLargeResult | `!getSnapshotP` | `!getSnapshotR`  | Browse |

Queries logs for the recorded signals at given point in time.

This method must be available on all nodes in the `.history/**` tree where
`getLog` is available.

The parameter is an *IMap* containing the following fields:

| Key | Name  | Type           |                        |
| --- | ----- | -------------- | ---------------------- |
| 1   | Time  | DateTime       | Defines point in time for log retrieval. |
| 2   | Ri    | [RPC Resource Identifier](../rpcri.md)\|Null | See `getLogP`. |

`!getSnapshotR` is defined as list of *IMap*s with following fields:

| Key | Name      | Type     |                     |
| --- | --------- | -------- | ------------------- |
| 1   | TimeStamp | DateTime | Timestamp of the record. This field is optional depending on ability of device to provide it. |
| 3   | Path      | String   | SHV path to the node relative to the path `getLog` was called on. *Default:* empty path `""`. |
| 4   | Signal    | String   | Signal name. *Default:* `chng`. |
| 5   | Source    | DateTime | Signal's associated method name.  *Default:* `get`. |
| 6   | Value     | RpcValue | Signal's value (parameter). |
| 7   | UserId    | String   | `UserId` carried by signal message. *Default:* `null`. |
| 8   | Repeat    | Bool     | `Repeat` carried by signal message. *Default:* `False`. |

The snapshot includes a list of records representing the state of every value
under the getLog SHV path at `Time`. This feature is primarily used to retrieve
the entire device state as it existed at the given moment. 

### `.history/**/.records/*`

These nodes provide systematic access to the records. Every record has unique ID
and mustn't change once it is recorded. This ID must be increasing for new
records but it doesn't have to be sequential (there can be unused IDs).

Other history implementations can take ID of their latest record and fetch
everything from it to correctly synchronize.

The implementation backed in not specified but records based access matches well
storage such as database or cyclic buffer.

#### `.history/**/.records/*:fetch`

| Name    | SHV Path                 | Flags           | Param Type                   | Result Type       | Access  |
|---------|--------------------------|-----------------|------------------------------|-------------------|---------|
| `fetch` | `.history/**/.records/*` | HintLargeResult | `[i:offset,i(0,):count]` | `!historyRecords` | Service |

This allows you to fetch records from log.

Parameter is tuple of first record ID to be provided and number of records (thus
last record returned is `parameter[0] + parameter[1] - 1`).

The call provides list of records. Every record is *IMap* with following fields:
* `0`(*type*): *Int* signaling record type:
  * `1`(*normal*) for normal records.
  * `2`(*keep*) for keep records. These are normal records repeated with newer
    date and time if there is no update of them in past number of records. They
    can also be used to seed log with values that are valid at first boot time
    (at log creation time).
  * `3`(*timeJump*) time jump record. This is information that all previous
    recorded times should actually be considered to be with time modification.
    The time offset is specified in field `60`. Field `1` must be also provided
    but others are not contrary to normal and keep records. This is recorded
    when time synchronization causes system clock to jump by more than a second.
  * `4`(*timeAbig*) time ambiguity record. This is information that date and
    time of the new logs has no relevance compared to the previous ones. Any
    subsequent records of type `3` should not be applied to them. This is
    recorded when time jump length can't be determined (backward skip of time
    commonly after boot) and thus time desynchronization is detected. The only
    field alongside this one must be `1`.
* `1`(*timestamp*): *DateTime* of system when record was created. This depends
  on record type. For normal records this is time of signal retrieval.
* `2`(*path*): *String* with SHV path to the node relative to the `.history`'s
  parent. The default if not specified is `""`.
* `3`(*signal*): *String* with signal name. The default if not specified is
  `"chng"`.
* `4`(*source*): *String* with signal's associated method name. The default if
  not specified is `"get"`.
* `5`(*value*): with signal's value (parameter). The default if not specified is
  `null`.
  specified is `null`.
* `6`(*accessLevel*): *Int* with signal's access level. The default if not
  specified is *Read*.
* `7`(*userId*): *String* with `UserId` carried by signal message. The default
  if not present is `null` and thus there was no user's ID in the message.
* `8`(*repeat*): *Bool* with `Repeat` carried by signal message. The default
  if not present is `false`.
* `60`(*timeJump*): *Int* with number of seconds of time skip. This is used with
  key `0` being `3`.

Fetch that is outside of the valid record ID range must not provide error.

#### `.history/**/.records/*:span`

| Name   | SHV Path                 | Flags  | Param Type | Result Type                         | Access  |
|--------|--------------------------|--------|------------|-------------------------------------|---------|
| `span` | `.history/**/.records/*` | Getter |            | `[i:smallest,i:biggest,i(1,):span]` | Service |

This allows fetch of boundaries for the record IDs and also the keep record
range.

This method provides three integers in a list. The first *Int* is the smallest
valid record ID, the second *Int* is the biggest valid record ID plus one (to
allow case when there are no records to be signaled with same value as the first
record) and the third *Int* is the keep record span.

The keep record span is range of records where all combinations of SHV path,
signal name and signal's associated method name in the log are present. That is
achieved by creating keep records that are copy of older ones when no signal for
them is received for some time. It allows of fetching the full log state without
going through the whole history.

### `.history/**/.files/*`

These nodes provide file based logs. The systematic log access is ensured by
only appending to the log files and never modifying them. Logs propagation is
then performed by copying appended data from existing files and new files.

Files are exposed as read only [file nodes](./file.md). The name of the file
must be date and time of the first record in the file in ISO-8601 format without
timezone and with seconds precision with ".log3" extension. The new files must
be created with date and time after the last log even if system clock is right
now set before that time. If system time is before the latest log file name then
latest log file name must be used with one second increased (to get unique file
name). The date and time recorded in the log file is still the system time, the
only modification here is the file name.

The file nodes must implement these methods: `.history/**/.files/*/*.log3:stat`,
`.history/**/.files/*/*.log3:size`, `.history/**/.files/*/*.log3:crc`,
`.history/**/.files/*/*.log3:read` and optionally
`.history/**/.files/*/*.log3:sha1`.

The content of the file logs is line separated CPON where initial line is
expected to have *Map* while rest should be *List*s.

The first line in the log file is *Map* with these fields:
* `"logVersion"` that right now should be set to `3.0` and thus must be
  *Decimal*.
* `"timeJump"`is the optional `true` or *Int* with the offset in seconds that
  should be applied to all log files before this one. This thus not specify
  offset of times in this file but times recorded in files so far. The `true` is
  used for ambiguity time jumps. This is used when time desynchronization is
  detected.

Notice that the first line is the only way to record time jump and thus when
time jump is detected you should always open a new log file.

The rest of the file must contain *List*s with following columns:
* *time*: *DateTime* of system when record was created or *Null* for anchor
  logs at the start of the file. File log must start with anchor logs of all
  latest recorded values from the previous log. This is to provide full
  information in a single log file.
* *path*: *String* with SHV path to the node relative to the `.history`'s parent.
  The default if not specified is `""`.
* *signal*: *String* with signal name. The default if not specified is `"chng"`.
* *source*: *String* with signal's associated method name. The default if not
  specified is `"get"`.
* *value*: signal's value (parameter). The default if not specified is `null`.
* *accessLevel*: *Int* with signal's access level. The default if not specified
  is *Read*.
* *userId*: *String* with `UserId` carried by signal message. The default if not
  present is `null` and thus there was no user's ID in the message.
* *repeat*: *Bool* with `Repeat` carried by signal message. The default if not
  present is `false`.

### `.history/**/.records/*:sync` and `.history/**/.files/*:sync`

| Name       | SHV Path                                           | Flags | Param Type | Result Type | Access       |
|------------|----------------------------------------------------|-------|------------|-------------|--------------|
| `lastSync` | `.history/**/.records/*` or `.history/**/.files/*` |       |            |             | SuperService |

Trigger the synchronization manually right now.

This can't be implemented for `.history/.records/*` and `.history/.files/*`
because those are logs not synchronized but collected.

This method triggers synchronization or does nothing if synchronization is
already in the progress.

### `.history/**/.records/*:lastSync` and `.history/**/.files/*:lastSync`

| Name       | SHV Path                                           | Flags  | Param Type | Result Type | Access  |
|------------|----------------------------------------------------|--------|------------|-------------|---------|
| `lastSync` | `.history/**/.records/*` or `.history/**/.files/*` | Getter |            | `t\|n`      | Service |

This provides information when last synchronization was performed.

This can't be implemented for `.history/.records/*` and `.history/.files/*`
because those are logs not synchronized but collected.

The provided value is either *DateTime* that must be when last synchronization
was performed and *Null* if synchronization is in progress right now.
Implementations should use time when they last called
`**/.history/**/.records/*:fetch` or `**/.history/**/.files/*:ls`.

## Time management in logs

Logs are recorded with device's UTC time. The RPC History then only copies these
logs from one instance to the other without modification. This means that date
and time is always kept as it was on the device that recorded it. This is ideal
when device has the correct real time clock but that might not be true and thus
time modifications come into play.

There are two types of time modifications recorded in the logs. We have either
known time jump or unknown time desynchronization.

The know time jump is detected on device when some log was already recorded and
suddenly system time doesn't correspond to the monotonic time since the last
record. The discrepancies up to 1 seconds should be disregarded and covered up
by time tweaking (that is because we record skips in seconds) to ensure that
time is still growing (no step backs in time compared to the previous log is
allowed unless time jump is recorded). This time jump happens commonly if some
tool synchronizes or in general updates system time. We expect that this
modification is always the correct one (that our time up to now was shifted by
skip) and time jump is recorded. `getLog` implementation thus must shift
virtually all recorded times. It can't modify date and time recorded in those
records but all previous records since the time jump up to the any time
desynchronization must be considered to be shifted by recorded time jump. The
multiple time jumps must be added together when you are reaching for older
records and have multiple time jumps in between.

The time desynchronization can happen only on first record after boot because
otherwise we know the previous time and can thus calculate the time jump. The
common detection for time desynchronization is the check of the latest record,
if it is in the future then desynchronization occurred and must be recorded.
Desynchronization creates break point in the time sequence and thus shifts
described in the previous paragraph are not performed after for records before
desynchronization. The only exception is in an unlikely event when logs after
desynchronization (after all time shifts from jumps are applied) are recorded as
happening before last log before desynchronization and in such case time shift
is introduced to move logs before desynchronization to be right before logs
after desynchronization. This can really happen only if someone sets date and
time in the future, then powers down the device, resets RTC and sets the correct
date and time after boot.

The optimal implementation of `getLog` for both records and files is to keep
index of modified times with reference to record ID or file with offset to speed
up lookup for `getLog`. The memory constrained devices can implement it in less
time optimal way by keeping only references to the time jumps and calculate the
correct time for every record when loaded. The logs time sequence is always kept
regardless of date and time and thus this time shifting only moves the whole
blocks of consistent logs.
