# linux-fuse-passthrough-patches

Linux kernel patches for the minimal `p4-min` FUSE passthrough write-parallelism fix.

This repo intentionally contains only the current submit-ready patch series and
supporting readme material. It replaces the older broader patch sequence that
included helper exports and follow-on contention work.

## Current Series

The current maintainer-facing series is a 2-patch minimal fix:

### [0001] fuse: preserve FOPEN_PARALLEL_DIRECT_WRITES for passthrough opens

Preserves `FOPEN_PARALLEL_DIRECT_WRITES` for opens that also carry
`FOPEN_PASSTHROUGH`.

Without this change, `fuse_file_io_open()` strips the flag before the
passthrough write path sees it, so passthrough writes remain serialized even if
userspace requested parallel direct writes.

### [0002] fuse: allow parallel direct writes in passthrough write_iter

Keeps the shared-vs-exclusive lock decision local to `fs/fuse/passthrough.c`.

The new logic allows shared inode locking only for safe direct overwrite I/O:

- `FOPEN_PARALLEL_DIRECT_WRITES` is present
- the write is direct I/O
- the write is not append
- the write does not extend past EOF

Buffered writes, append writes, and EOF-extending writes remain serialized.

## Not Part Of This Minimal Series

The following older patch ideas are intentionally excluded from the current
submit-ready series:

- exporting `fuse_dio_lock()` from `fs/fuse/file.c`
- exporting `fuse_dio_unlock()` from `fs/fuse/file.c`
- adding declarations in `fs/fuse/fuse_i.h`
- the broader `iocachectr` / `fi->lock` follow-on cleanup work

Those may still be interesting separately, but they are not required for the
minimal upstreamable `p4-min` correctness fix.

## Validation Snapshot

Validated on kernel `6.17.13-p4min` with fio 4K random writes on a FUSE
passthrough mount backed by XFS on a RAM-backed null_blk device:

| numjobs | FUSE IOPS |
|---------|-----------|
| 1       | 481,837   |
| 2       | 931,883   |
| 4       | 1,533,058 |
| 8       | 1,730,477 |

Raw XFS baseline at `numjobs=8`: `1,693,406 IOPS`.
