# pipa HOS2 memory-map audit

Status: evidence review only. No runtime memory-map changes are proposed by this note.

## Scope

Compare the HOS2 `uefiplatLA.cfg` extracted from the supplied `xbl_a` with the repository's
`Platforms/Xiaomi/pipaPkg/Library/MemoryMapLib/MemoryMapLib.c`.

The HOS2 config is useful device-specific evidence, but it is not internally consistent in
all of the ranges below. Do not copy its DDR map wholesale until those conflicts are resolved.

## Confirmed conflicts / differences

### 1. Low-memory reservation and IPC SHM

- HOS2 `UnusableDDRMemoryStartAddr`: `0x80000000`
- HOS2 `UnusableDDRMemorySizeAtBeginning`: `0x00600000`
- This covers `[0x80000000, 0x80600000)`.
- HOS2 `IPC SHM`: `[0x805D0000, 0x805F0000)`.

The IPC SHM entry overlaps the stated unusable-at-beginning interval by `0x20000` bytes.
The current repository's `HYP` entry also reserves `[0x80000000, 0x80600000)`.
Do not simply add IPC SHM as a separate descriptor without authoritative evidence about
the intended reservation/HOB semantics.

### 2. GPU PRR and XBL Log Buffer

- Repository `GPU PRR`: `[0x80880000, 0x80890000)`
- HOS2 `XBL Log Buffer`: `[0x80884000, 0x80894000)`

The ranges overlap over `[0x80884000, 0x80890000)`, a length of `0xC000` bytes.
The HOS2 entry labels XBL Log Buffer as system memory with write-back attributes, while
GPU PRR is a memory-reserved region with different attributes. Splitting or retyping either
range without a board-specific source would be speculative.

### 3. PIL / ADSP reserved ranges

- HOS2 `PIL Reserved`: `[0x86000000, 0x93200000)`
- Repository `PIL Reserved`: `[0x86200000, 0x92700000)`
- Repository `ADSP RPC`: `[0x92700000, 0x92F00000)`

The repository's two entries do not cover the same range as the single HOS2 PIL reservation:
the repository starts 2 MiB later, and its combined PIL + ADSP span ends 3 MiB before the
HOS2 reservation. This is a descriptor/semantics difference, not enough evidence to safely
rewrite the entries.

### 4. Additional HOS2 entries absent from the repository DDR list

The HOS2 map contains entries for MPSS EFS, BOOT INFO, Sched Heap, DBI Dump, FV Region,
ABOOT FV, SEC Heap, CPU Vectors, MMU PageTables, Log Buffer, and Kernel that are not
currently listed as DDR descriptors in the repository's map. Some may be accounted for by
other firmware components or intentionally omitted; presence in the source config alone
does not establish that every entry should be duplicated in this library.

### 5. Existing agreement

The following HOS2/repository DDR ranges match by base and size: AOP CMD DB
(`0x80860000 + 0x20000`), SMEM (`0x80900000 + 0x200000`), TZApps
(`0x82400000 + 0x3A00000`), DXE Heap (`0x98900000 + 0x3300000`),
Display Reserved (`0x9C000000 + 0x2400000`), UEFI FD (`0x9FC00000 + 0x300000`),
UEFI Stack (`0x9FF90000 + 0x40000`), and Info Blk (`0x9FFFF000 + 0x1000`).
The previously inspected 23 RegisterMap ranges also match by base and size.

## Recommendation

Keep `MemoryMapLib.c` unchanged for now. Resolve the low-memory and GPU/XBL overlaps from
an authoritative pipa platform source (or a validated boot log/HOB dump) before changing
addresses, lengths, memory types, cache attributes, or HOB options. Then make a small patch
with an explicit before/after range table and run the build plus static overlap checks.

This audit is not a bootability claim and does not authorize flashing or writing device
partitions.
