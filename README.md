Go casync
=========

This project doesn't seek to be a full reimplementation of upstream casync, but is a collection of tools that utilize certain aspects of casync for very specific use cases.

## Goals And Non-Goals

Among the distinguishing factors:

- Where the upstream command has chosen to optimize for storage efficiency (f/e, being able to use local files as "seeds", building temporary indexes into them), this command chooses to optimize for runtime performance (maintaining a local explicit chunk store, avoiding the need to reindex) at cost to storage efficiency.
- Where the upstream command has chosen to take full advantage of Linux platform features, this client chooses to implement a minimum featureset and, while high-value platform-specific features (such as support for btrfs reflinks into a decompressed local chunk cache) might be added in the future, the ability to build without them on other platforms will be maintained.

- Supporting the .catar archive format (whether or not packaged in a .caidx) is not presently a goal.
- SHA512/256 is currently the only supported hash function.
- Only chunk store using zstd compression are supported at this point.

## Tools

### desync

Basic client to read casync blob index files (caibx) and reassemble a blob from chunks read from a chunk store (local or remove via SSH).

#### Options
- `-s <store>` Location of the chunk store, can be local directory or a URL like ssh://hostname/path/to/store
- `-c <store>` Location of a *local* chunk store to be used as cache. Needs to be writable.
- `-n <int>` Number of concurrent download jobs and ssh sessions to the chunk store.

#### Environment variables
- `CASYNC_SSH_PATH` overrides the default "ssh" with a command to run when connecting to the remove chunk store
- `CASYNC_REMOTE_PATH` defines the command to run on the chunk store when using SSH, default "casync"

#### Caching
The `-c <store>` option can be used to either specify an existing local store to act as cache or to build a new one. Whenever a cache is requested, it is first looked up in the local cache before routing the request to the main (likely remote store). Any chunks downloaded from the main store are added to the local store (cache). In addition, when a chunk is read from the cache, mtime of the chunk is updated to allow for basic garbage collection based on file age. The cache directory as well as the chunks in it are expected to be writable. If the cache contains an invalid chunk (checksum does not match the chunk ID), blob assembly will fail. Invalid chunks are not skipped or removed from the cache automatically.

#### Examples:

Re-assemble somefile.tar using a remote chunk store and a blob index file.
```
desync -s ssh://192.168.1.1/path/to/casync.store/ -c /tmp/store somefile.tar.caibx somefile.tar
```

### verify-store

This tool is mainly used to clean a local cache store that may have bad chunks in it. It goes through all chunks in the store and compares the checksums to the expected ones. By default, it only reports bad chunks. If the `-r` is provided, bad chunks are removed from the store.

#### Options

- `-r` Remove any chunks that contain data not matching the chunk ID. Use with caution.
- `-n <int>` Number of concurrent verification jobs. Higher numbers increase CPU and disk I/O. Default 10.

#### Examples:

Report any invalid chunks in a local cache. Use 5 goroutines.
```
verify-store -n 5 /path/to/cache
```

## TODOs
- Write tests
- Pre-allocate the output file to avoid fragmentation
- Check output file size, compare to expected size
- Support retrieval of index files from the chunk store
- Allow on-disk chunk cache to optionally be stored uncompressed, such that blocks can be directly reflinked (rather than copied) into files, when on a platform and filesystem where reflink support is available.
- When using the remote store, multiple SSH sessions and csync processes are started, there's nothing to stop them yet (relies on process shutdown/cleanup)
- Code cleanup and reorg
- Support HTTP for remote stores
- Support multiple stores as source