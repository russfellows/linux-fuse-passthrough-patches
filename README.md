# linux-fuse-passthrough-patches

Linux kernel patches fixing `FOPEN_PARALLEL_DIRECT_WRITES` for FUSE passthrough mounts.

Submitted to `linux-fsdevel@vger.kernel.org` on 2026-05-29.  
Thread: https://lore.kernel.org/linux-fsdevel/20260529031918.7361-1-russ.fellows@gmail.com/

## Patches

### [0001] fuse: fix FOPEN_PARALLEL_DIRECT_WRITES being ignored for passthrough writes

Fixes two bugs that prevent `FOPEN_PARALLEL_DIRECT_WRITES` from having any effect on
FUSE passthrough mounts:

1. `fuse_passthrough_write_iter()` called `inode_lock()` directly, bypassing the
   `fuse_dio_lock()` function that checks the flag.
2. `fuse_file_io_open()` stripped `FOPEN_PARALLEL_DIRECT_WRITES` from any open lacking
   `FOPEN_DIRECT_IO`, including passthrough opens where the flag is valid without it.

`Fixes: 4d99ff8f6b85 ("fuse: implement open/create with FOPEN_PASSTHROUGH")`  
`Cc: stable@vger.kernel.org`

### [0002] fuse: reduce fi->lock contention on parallel direct I/O

Converts `iocachectr` from `int` to `atomic_t` and adds lockless fast paths to
`fuse_inode_uncached_io_start/end` and `fuse_write_update_attr`, eliminating
~5.1M spinlock acquisitions/second at 1.7M IOPS.

## Results

fio randwrite 4K direct, iodepth=64, FUSE passthrough on XFS/null_blk, kernel 6.17.13:

| numjobs | Before (IOPS) | After patch 1 | After patch 2 |
|---------|---------------|---------------|---------------|
| 1       | ~470K         | ~470K         | ~470K         |
| 2       | ~650K         | ~930K         | ~930K         |
| 4       | ~640K         | ~1,530K       | ~1,530K       |
| 8       | ~635K         | ~1,707K       | ~1,707K       |

Raw XFS baseline at numjobs=8: ~1,702K IOPS.
