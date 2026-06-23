# wxl-modern-wmo

Loads modern world map objects (WMO) in the Client by down-converting them, in memory, into the shape the
native WMO loader reads. No files are written and no native WMO is touched.

## What it does

A WMO ships as a root file (`<name>.wmo`) plus N group files (`<name>_NNN.wmo`); the client opens each one
on its own. Modern WMOs add chunks the native loader has no slot for and encode some fields differently.
The native loader walks chunks **positionally** (it assumes the canonical order and advances by each
chunk's size), so an unknown chunk left in place desynchronizes the whole parse.

On each `.wmo` open the host decides root vs group from the chunk after `MVER` (`MOHD` = root, `MOGP` =
group) and reshapes the bytes the host already read:

- **Root**: strips the source-only chunks (`GFID`/`MOUV`/`MODI`/`MOSI`), rebuilds `MOTX` (resolving any
  texture FileDataID references), collapses the material shader family down to the Client's `0..6`, and
  clears the runtime texture-handle bytes the Client expects zero on disk.
- **Group**: keeps the chunks the Client knows in canonical order (synthesizing the mandatory ones when
  absent), converts `MPY2` back to `MOPY`, relocates the `MOBA` material id and rebuilds its bounding box,
  and **recomputes the group flags** the loader gates sub-chunk consumption on while preserving every
  ungated bit.

The transform is **data-gated** (it keys off chunk presence and field values, not a version number), so
one path serves any source. A file already Client-shaped is served unchanged.

Sources that name their textures directly (a `MOTX` name table) need no resolver. Sources that reference
textures by FileDataID rebuild the table from a resolver callback (not wired yet).

## Layout

- `shared/` — the byte transform, compiled into both the host and the DLL:
  - `WmoTranslate.{hpp,cpp}` — root + group down-convert.
  - `WmoClient.hpp` / `WmoSource.hpp` / `WmoChunks.hpp` — the Client contract, the source additions, and
    the per-magic disposition policy.
  - `ChunkIO.hpp` — generic IFF chunk walker.
  - `Resolver.hpp` — the FileDataID -> path callback contract.
- `host/WmoHost.cpp` — registers the serve-time transform and dispatches root vs group.

The runtime side (defensive draw/portal/group guards that wrap the native WMO render path, and the eventual
multi-pass overlay) lives in the engine core, not in this module — the same split the native M2 support
uses.

## Building

The transform runs host-side, so build the host alongside the DLL:

```
.\build.ps1 -BuildHost
```
