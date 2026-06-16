# linux-fuse-passthrough-patches

Linux kernel patches for the FUSE passthrough write-parallelism fix.

This repo tracks the current submit-ready patch series (v3) along with the full
source of the four modified kernel files for easy review.

## Repository Layout

```
patches/   — mbox-format patch files, ready for git send-email
src/       — full source of the four modified kernel files (for review context)
  fs/fuse/
    iomode.c
    passthrough.c
    file.c
    fuse_i.h
```

## Current Series: v3

Submitted 2026-06-16 to `miklos@szeredi.hu`, `fuse-devel@lists.linux.dev`,
`linux-fsdevel@vger.kernel.org`, `linux-kernel@vger.kernel.org`.
Cc: Amir Goldstein `<amir73il@gmail.com>`.

### Changes since v2

- **Patch 1**: Introduce `FOPEN_IOMODE` helper macros (`FOPEN_IOMODE_IS_CACHED`,
  `FOPEN_IOMODE_IS_DIRECT`, `FOPEN_IOMODE_IS_PASSTHROUGH`) to make the flag
  conditions self-documenting. Use `FOPEN_IOMODE_IS_CACHED` for the
  `PARALLEL_DIRECT_WRITES` guard. No behavioral change.
- **Patch 2**: Replace open-coded lock/unlock with `fuse_passthrough_lock()` and
  `fuse_passthrough_unlock()` helpers local to `passthrough.c`. Add a post-lock
  re-check of the past-EOF condition to close the TOCTOU window between the
  initial check and acquiring the shared inode lock. Restore
  `fuse_dio_lock()`/`fuse_dio_unlock()` to file-private (static) and remove
  their declarations from `fuse_i.h`.

### [0001] fuse: preserve FOPEN_PARALLEL_DIRECT_WRITES for passthrough opens

`fuse_file_io_open()` was stripping `FOPEN_PARALLEL_DIRECT_WRITES` for any open
lacking `FOPEN_DIRECT_IO`. That rule is correct for the regular direct-IO path
but wrong for passthrough: passthrough already bypasses the page cache through
the backing file without needing `FOPEN_DIRECT_IO`. The new
`FOPEN_IOMODE_IS_CACHED` guard expresses the correct invariant: suppress the
flag only for cached (page-cache) I/O mode.

### [0002] fuse: allow parallel direct writes in passthrough write_iter

`fuse_passthrough_write_iter()` held the exclusive inode lock unconditionally,
ignoring `FOPEN_PARALLEL_DIRECT_WRITES` entirely.

The new `fuse_passthrough_lock()` allows shared inode locking only for safe
direct overwrite I/O:

- `FOPEN_PARALLEL_DIRECT_WRITES` is set
- the write is direct I/O (`IOCB_DIRECT`)
- the write is not append (`!IOCB_APPEND`)
- the write does not extend past EOF (post-lock TOCTOU re-check)

Buffered writes, append writes, and EOF-extending writes remain serialized with
an exclusive lock. Passthrough files are always in uncached iomode so the
`fuse_inode_uncached_io_start()` guard from `fuse_dio_lock()` is not needed.

## Not Part Of This Minimal Series

- exporting `fuse_dio_lock()` / `fuse_dio_unlock()` from `fs/fuse/file.c`
- adding declarations in `fs/fuse/fuse_i.h`
- the broader `iocachectr` / `fi->lock` follow-on cleanup work

## Performance

Tested on kernel `6.17.13-p4min-v3`, custom FUSE daemon advertising
`FOPEN_PASSTHROUGH | FOPEN_PARALLEL_DIRECT_WRITES`, XFS on a RAM-backed
null\_blk device. fio 4K random write, `iodepth=64`:

| numjobs | FUSE IOPS  | Raw XFS IOPS |
|---------|------------|--------------|
| 1       | 478,564    | 528,835      |
| 2       | 941,483    | ~1.0M        |
| 4       | 1,457,284  | ~1.5M        |
| 8       | 1,675,749  | 1,693,406    |

FUSE at `numjobs=8` reaches 99% of raw XFS throughput.

## Correctness Testing

All seven test cases passed on kernel `6.17.13-p4min-v3+` with a custom FUSE
daemon advertising `FOPEN_PASSTHROUGH | FOPEN_PARALLEL_DIRECT_WRITES`, backed
by XFS on a RAM-backed null\_blk device (8 GiB):

| Case          | What it validates                                              | Result |
|---------------|----------------------------------------------------------------|--------|
| `overwrite`   | Parallel shared-lock writes, CRC32C per-block verify          | PASS   |
| `read_write`  | Concurrent verified writers + unlocked readers, no deadlock   | PASS   |
| `append`      | Multi-job pwrite within pre-allocated EOF (shared lock path)  | PASS   |
| `o_append`    | `O_APPEND` / `IOCB_APPEND` forces exclusive lock, atomicity   | PASS   |
| `extend`      | Past-EOF disjoint writes force exclusive via TOCTOU re-check  | PASS   |
| `extend_race` | N jobs racing to same past-EOF region, no deadlock            | PASS   |
| `buffered`    | Buffered writes bypass passthrough path entirely (negative)   | PASS   |
