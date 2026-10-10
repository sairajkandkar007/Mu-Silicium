# pipa HOS2 memory-map review

## Current proposal

The branch now applies a community-reported pipa memory map from
[Project-Silicium/Mu-Silicium issue #2145](https://github.com/Project-Silicium/Mu-Silicium/issues/2145).
In that issue, a maintainer/helper posted this layout as one that had worked for another pipa
user, and the issue reporter later confirmed their own boot problem was resolved. This is
useful device-specific evidence, but it is a community report, not a formal platform specification
or a test of this repository's exact HOS2 firmware/build.

Changes in `MemoryMapLib.c`:
- Change `HYP` resource type from `SYS_MEM` to `MEM_RES`, retaining the same range.
- Add `QTEE` reservation at `0x80B00000 + 0x01900000`.
- Change `TZApps` HOB option to `AddMem`, retaining its range and cache attributes.
- Move `DXE Heap` from `0x98900000 + 0x03300000` to `0x92F00000 + 0x09100000`.

The DXE heap proposal fills `[0x92F00000, 0x9C000000)`, ending exactly where
`Display Reserved` begins. The `PIL Reserved` entry ends at `0x92700000`, leaving an
8 MiB gap before the DXE heap starts. The new QTEE range ends at `0x82400000`, where
TZApps begins. The low-memory reservation remains `[0x80000000, 0x80600000)`; the
internally overlapping HOS2 `IPC SHM` entry is not added as a separate descriptor. Likewise,
the overlapping HOS2 `XBL Log Buffer` entry is not added over GPU PRR.

## Source HOS2 config discrepancies

The extracted `uefiplatLA.cfg` is not fully self-consistent as a standalone map:
- `UnusableDDRMemoryStartAddr=0x80000000` and size `0x00600000` cover
  `[0x80000000, 0x80600000)`, overlapping HOS2 `IPC SHM` at
  `[0x805D0000, 0x805F0000)`.
- HOS2 `XBL Log Buffer` at `[0x80884000, 0x80894000)` overlaps the repository's
  GPU PRR range `[0x80880000, 0x80890000)`.
- HOS2 PIL and the repository's PIL/ADSP ranges differ.

Because of these conflicts, the branch uses the reported working pipa map rather than blindly
copying the HOS2 config's `MemoryMap` entries.

## Static range review

For the changed DDR descriptors, the proposed boundaries are adjacent or separated:
- QTEE: `[0x80B00000, 0x82400000)`
- TZApps: `[0x82400000, 0x85E00000)`
- PIL Reserved: `[0x86200000, 0x92700000)`
- DXE Heap: `[0x92F00000, 0x9C000000)`
- Display Reserved: `[0x9C000000, 0x9E400000)`

No new overlap is apparent in these changed ranges. This is a static address-boundary review only;
it does not validate HOB semantics, cacheability, DXE allocation behavior, device boot, or
Windows/Linux compatibility.

## Remaining validation

1. Review the proposed diff before merging.
2. Build the platform and inspect build reports/logs.
3. Validate memory descriptors and HOB output with a safe, non-destructive test setup.
4. Do not treat a successful build as proof that the tablet will boot. Do not flash or write
   device partitions based only on this source review.
