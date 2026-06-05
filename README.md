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

## Current State

**Proven.** Tested via spl6 POC probe (`hyperdrive-fuse-readonly`) in Docker
with `--cap-add SYS_ADMIN --device /dev/fuse`:

- readdir: root + subdirectories, correct entries
- read: file content through the mount matches Hyperdrive content
- stat: file/directory detection, correct sizes
- write rejection: EROFS returned
- clean mount/unmount cycle

Key finding: FUSE handlers must be async-compatible. Synchronous fs calls from
the consumer side (e.g. `readdirSync`) deadlock the Node.js event loop — the
FUSE callback needs the loop to resolve Hyperdrive's async operations.

Runs under Node.js, not Bare. `fuse-native` requires Node builtins (os, fs,
child_process). This is appropriate — the FUSE mount is the OS bridge by nature.

FUSE is built into the WSL2 kernel — works on WSL2 without a custom kernel.
Mounted drives are accessible from Windows via `\\wsl$\`.

**Not yet exercised:** large directories, deep nesting, concurrent reads,
symlinks, performance under load, multi-peer replication while mounted.

## Intention

The visible face of the P2P platform. Joining a swarm = mounting a FUSE drive.
The MVP is read-only — the drive is a catalogue of swarm content. Apps on the
drive execute and connect to the swarm natively for data operations.

Future: writable workspaces (per-node, path-based permissions), app startup
from the mounted drive, integration with the mycelium module for fabric
operations.
