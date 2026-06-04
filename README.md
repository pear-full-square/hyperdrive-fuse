# hyperdrive-fuse

FUSE mount for Hyperdrive v11. Mount a P2P drive as a local filesystem.

## MVP — read-only

The drive is a window into the swarm, not a writable surface. All write
operations return EROFS (read-only filesystem). Writes go through the
fabric (`spl` commands or native Hyperdrive API), not through the mount.

## Usage

### Docker (recommended)

Build:
```
docker build -t hyperdrive-fuse .
```

Interactive — mount and browse:
```
# Terminal 1: start the mount
docker run -it --name fuse-demo --cap-add SYS_ADMIN --device /dev/fuse \
  hyperdrive-fuse index.mjs

# Terminal 2: browse the mounted drive
docker exec -it fuse-demo bash
ls /tmp/fuse-mnt/
cat /tmp/fuse-mnt/hello.txt
find /tmp/fuse-mnt
```

Run the test:
```
docker run --rm --cap-add SYS_ADMIN --device /dev/fuse hyperdrive-fuse test.mjs
```

### Requirements

- FUSE kernel support (`--cap-add SYS_ADMIN --device /dev/fuse` in Docker)
- Node.js (the FUSE mount is the OS bridge — runs under Node, not Bare)
- libfuse2 (`apt install fuse`)

## FUSE handlers implemented

| Operation | Behaviour |
|---|---|
| readdir | List Hyperdrive directory entries |
| getattr | File/directory stat (size, mode) |
| open | Read-only — rejects write flags |
| read | Read file content from Hyperdrive |
| release | Close file handle |
| write, create, unlink, mkdir, rmdir, rename | EROFS |

## Architecture

```
OS filesystem calls (ls, cat, find)
    ↓ FUSE kernel module
fuse-native (Node.js)
    ↓ FUSE handlers
Hyperdrive v11 (get, entry, readdir)
    ↓
Hypercore / Corestore / RocksDB
```

The mount is pure visibility. The swarm is where the work happens.
