# History Provider 3
_History Provider 3_ (HP3) is a program whose goal is to gather logs from the shv tree and provide a `getLog` interface.

To do that _HP3_ does these things:
- retrieves logs from other sources
- exposes a [`getLog`](#historyshvgetlog) interface for every site
- exposes a [`_shvjournal`](#history_shvjournal) node, which serves as a controlling node and also as a container for every file that _HP3_ has
  under its possession

## Startup
At startup, _HP3_:
- Retrieves sites definition via `sites:getSites`.
- Builds an shv tree of all the sites under the [`history/`](#shv-tree) root node according to the sites. The same tree is also
  created as a filesystem directory tree.

This process can be repeated even after startup via [`history:reloadSites`](#historyreloadsites)

## HP3 sites definition
_HP3_ scans sites for `HP` and `HP3` nodes. A `HP` node signifies that _HP3_ can download logs through
[`getLog`](#historyshvgetlog). A `HP3` node signifies that _HP3_ can download logs through file synchronization. This a
JSON schem for a meta.json file containing a `HP3` node:
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Elektroline HP3 metadata",
  "type": "object",
  "properties": {
    "$schema": {
      "type": "string"
    },
    "HP3": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "pattern": "^device|HP3$",
          "default": "default"
        },
        "syncPath": {
          "type": "string",
          "pattern": "^.app/history|.app/shvjournal$",
          "default": ".app/history"
        }
      },
      "additionalProperties": false
    }
  }
}
```
An `HP3` node can be either `device` (which is the default) or `HP3`.
A `device` node also specificies a `syncPath`, i.e. where on the device should _HP3_ look for files. By default, it is
`.app/history`. Older,  `.app/shvjournal` is also supported.

_HP3_ stops at the first `HP`/`HP3` node it encounters and downloads logs from that node. The `HP3` type signifies that
the node is a _HP3_ and its logs includes all logs from its subtree. This creates a cascade so that logs gradually flow
from the `device`s (_shvgate_) to the top _HP3_.

## How does _HP3_ sync logs?
There three ways _HP3_ can obtain logs:

0) Capturing events.
1) Downloading files via "file synchronization".
2) Downloading logs via `getLog` and saving them to files.

If the log directory is empty for the specific site, _HP3_ only downloads history up until a specific point (controlled
by the configuration). Afterwards, only new logs are ever downloaded, _HP3_ never downloads logs, that are older than it
already has

### Capturing events
This is the most straightforward way of getting logs and isn't really "downloading". _HP3_ subscribes all the sites for
`chng` events and saves them to a file called dirtylog. The dirtylog serves as a temporary storage of logs, until actual
logs are retrieved from other sources. When new logs are retrieved, entries that are older than the newest synced entry
are removed. This process is called "dirty log trimming".

### File synchronization
The standard way for retrieving logs is via file synchronization. _HP3_ retrieves a file list from a node and checks the
difference between the local database and the remote database. All newer files are synced to the local database. The
filelist can be provided by either:
- a "device" (e.g. _shvgate_) - filelist should be in `<site-path>/.app/shvjournal`
- HP3 - filelist should be in `history/_shvjournal`
`shvjournal` or `_shvjournal`. The location of the filelist node (and the file nodes) can be found through the sites
definition.

### Downloading via `getLog`
Devices that don't support directly retrieving files, _HP3_ falls back to retrieving logs via `getLog`. _HP3_ calls the
method, figures out a filename and saves that to the disk. The filenames are more or less guessed, so they won't
correspond to the filenames on the device.

## When does _HP3_ sync logs?
### Manually through the [`syncLog`](#history_shvjournalsynclog). method
The user can invoke the [`syncLog`](#history_shvjournalsynclog) method to manually sync sites. More info the SHV tree
section.

### Periodically over time
_HP3_ iterates over all sites and syncs them periodically one by one. The interval between each site is configurable
(default one minute).

### On the `mntchng` signal
Whenever a site gets a `mntchng` signal, it might mean that it has been offline for some time, and our data is stale,
because we couldn't get events for our dirty log. Therefore, the site gets synced.

## Sanitizer
_HP3_ periodically iterates over all sites' log directories and removes old files from them. The total size of all logs
is configurable and it is split evenly over all the sites. The interval between sanitizing is also configurable (default
one minute).

## SHV tree
### `history:reloadSites`
| Name          | SHV Path   | Signature   | Flags  | Access |
|---------------|------------|-------------|--------|--------|
| `reloadSites` | `history/` | `ret(void)` |        | Write  |
Re-initializes the SHV tree:
1) Fetches new sites definition via `sites:getSites`.
2) Creates the SHV tree according to the new sites.

### `history/_shvjournal`
This is where _HP3_ exposes its saved log files for other HP3s to download from. See
[shvjournal](./rpcmethods/shvjournal.md) for details on how the files are served.

### `history/_shvjournal/*:syncLog`
| Name      | SHV Path                | Signature    | Flags  | Access |
|-----------|-------------------------|--------------|--------|--------|
| `getLog`  | `history/shv/*:syncLog` | `ret(param)` |        | Write  |

Triggers a manual sync on a subset of sites.

| Parameter                                    | Result     |
|----------------------------------------------|------------|
| `String`                                     | `String[]` |
| `{"waitForFinished":Bool, "shvPath":String}` | `String[]` |

The method takes a mandatory string argument to match the subset, it mustn't be an empty string. Specifically:
- a string which is matched against the name of the site and only matching sites are synced, or
- a map agument with these keys:
   - waitForFinished?: `Bool` - if true, this method will wait for the completion of the syncing before retuning a
     result
   - shvPath: `String`: filter used for matching the subset of sites

The return value is an array of the matched sites.

### `history/_shvjournal/*:syncInfo`
| Name      | SHV Path                 | Signature    | Flags  | Access |
|-----------|--------------------------|--------------|--------|--------|
| `getLog`  | `history/shv/*:syncInfo` | `ret(param)` |        | Read   |

Returns info about the last sync process for sites. The method takes a string argument which is matched against the name
of the site and then only matching sites are synced.

| Parameter | Result                    |
|-----------|---------------------------|
| `String`  | `{<site-path>: String[]}` |

### `history/shv/<path-to-site>`
Every site path has a few methods implemented. The main interface for HP3 is the `getLog` method

### `history/shv/*:getLog`
| Name      | SHV Path               | Signature    | Flags  | Access |
|-----------|------------------------|--------------|--------|--------|
| `getLog`  | `history/shv/*:getLog` | `ret(param)` |        | Read   |

### `history/shv/*:logSize`
| Name       | SHV Path                 | Signature   | Flags  | Access |
|------------|--------------------------|-------------|--------|--------|
| `logSize`  | `history/shv/*:logSize`  | `ret(void)` | Getter | Read   |

Returns disk space usage by this site in bytes.

### `history/shv/*:sanitizeLog`
| Name           | SHV Path                     | Signature   | Flags  | Access |
|----------------|------------------------------|-------------|--------|--------|
| `sanitizeLog`  | `history/shv/*:sanitizeLog`  | `ret(void)` |        | Write  |

Manually run the sanitizer on this site.
