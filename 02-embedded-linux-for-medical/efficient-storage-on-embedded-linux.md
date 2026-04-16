# Efficient Storage Design on Embedded Linux

This document summarises practical lessons about designing low-level, append-only storage on embedded Linux systems
backed by eMMC flash. The patterns here apply broadly to any time-series or log-structured data store that needs to
survive power loss, minimise flash wear, and remain efficient with constrained resources.

---

## Table of Contents

1. [Storage Stack Fundamentals](#storage-stack-fundamentals)
2. [Page-Aligned Writes with pwrite()](#page-aligned-writes-with-pwrite)
3. [Crash Durability with fsync()](#crash-durability-with-fsync)
4. [Advisory Writeback Scheduling](#advisory-writeback-scheduling)
5. [Space Pre-allocation with fallocate()](#space-pre-allocation-with-fallocate)
6. [Reclaiming Space with ftruncate()](#reclaiming-space-with-ftruncate)
7. [eMMC Flash Characteristics](#emmc-flash-characteristics)
8. [Segment-Based File Layout](#segment-based-file-layout)
9. [Self-Describing Recovery Without a Journal](#self-describing-recovery-without-a-journal)
10. [Durability Trade-offs](#durability-trade-offs)
11. [Summary Checklist](#summary-checklist)

---

## Storage Stack Fundamentals

Understanding what guarantees each layer provides is essential before designing a storage module.

```text
Application code
     ↓  read / write / fsync system calls
Kernel VFS + Page Cache
     ↓  dirty pages flushed to block layer
Block layer + I/O scheduler
     ↓  requests merged, reordered
eMMC FTL (Flash Translation Layer)
     ↓  logical → physical mapping, wear leveling
NAND flash cells
```

Key takeaways:

- **Writes are not synchronous by default.** `write()` and `pwrite()` place data in the kernel page cache. The kernel
  writes it to storage later, in the background.
- **Readers in the same OS see the page cache.** A concurrent reader will see data written by `pwrite()` without waiting
  for it to reach flash — the shared page cache is the source of truth for both sides.
- **`fsync()` is the only portable guarantee of crash durability.** It blocks until the storage device acknowledges that
  all dirty pages for the file descriptor have been committed to non-volatile storage.

---

## Page-Aligned Writes with pwrite()

### What it does

`pwrite(fd, buf, count, offset)` writes `count` bytes at `offset` without moving the file position. POSIX guarantees
that it is atomic with respect to other `pwrite()` calls on the same file.

### Why alignment matters on flash

Flash storage operates on fixed physical pages (typically 4 KB on eMMC). A write that is smaller than a page, or
misaligned to a page boundary, forces the FTL to perform a **read-modify-write (RMW)** cycle:

1. Read the entire physical page into a buffer
2. Patch the bytes that changed
3. Erase the page (or the enclosing erase block)
4. Write the patched page back

RMW cycles consume more time, more write endurance, and increase write amplification. Aligning every write to a multiple
of the filesystem block size (4 KB for most Linux filesystems) avoids them entirely.

### Practical rule

Choose a write unit (`PageSize`) equal to the filesystem block size — 4 KB is almost universally correct. Accumulate
records in memory until a full page is ready, then issue a single aligned `pwrite()`.

### Concurrent visibility guarantee

Once `pwrite()` returns, any other process or thread that reads the same byte range via `read()` or `pread()` will see
the new data. No flush or `fsync()` is required for reader visibility — just for crash durability.

---

## Crash Durability with fsync()

### What it does

`fsync(fd)` flushes all dirty data and metadata for `fd` from the kernel page cache to the storage device and waits for
the device to confirm the write. After `fsync()` returns, the data survives a sudden power loss.

### The cost

On eMMC, `fsync()` is expensive:

- The kernel flushes all dirty pages for the file to the block layer.
- The eMMC device drains its internal write cache and commits pages to NAND cells.
- On some devices this takes 5–50 ms per call.
- Frequent `fsync()` calls shorten flash lifetime by forcing more program/erase (P/E) cycles.

### Design implication

Avoid calling `fsync()` after every record or every page. Instead, batch data and call `fsync()` at meaningful
checkpoints — for example, when an entire fixed-size segment is complete. This amortises the cost across many records
while still bounding the data loss window.

### Metadata durability

`fsync()` on a file does not guarantee that a newly created file is visible after a crash — the directory entry also
needs to be flushed. Call `fsync()` on the directory file descriptor after creating a new file if visibility of the file
itself must survive a crash.

---

## Advisory Writeback Scheduling

### sync_file_range()

`sync_file_range(fd, offset, nbytes, flags)` is a Linux-specific call that hints to the kernel to start writing dirty
pages in a range to the block layer _without blocking the caller_. It does **not** call `fsync()` on the device and
provides **no crash-durability guarantee**.

With `SYNC_FILE_RANGE_WRITE`, the kernel begins scheduling writeback for the indicated range but returns immediately.
This is useful for:

- Spreading I/O over time so that a later `fsync()` has less work to do (reducing the spike in write latency).
- Keeping dirty page pressure low on memory-constrained devices.

Do not use `sync_file_range()` as a substitute for `fsync()`. Data in the kernel page cache has not yet been committed
to NAND and will be lost on power failure.

---

## Space Pre-allocation with fallocate()

### Problem: fragmentation and write amplification

When a file grows one block at a time, the filesystem allocates blocks on demand. On fragmented filesystems, this can
scatter the file's blocks across the storage medium. On flash, it forces the FTL to work harder during garbage
collection, increasing write amplification.

### Solution: fallocate()

`fallocate(fd, 0, offset, len)` asks the filesystem to reserve `len` bytes of contiguous space from `offset`. The
reserved space is immediately part of the file (file size grows).

For cases where the caller wants to reserve space without changing the visible file size, use:

```c
fallocate(fd, FALLOC_FL_KEEP_SIZE, offset, len);
```

This pre-allocates physical blocks so subsequent writes do not need to allocate on the fly, but readers see the same
file size as before.

### Choosing the allocation chunk size

Align the allocation size to the flash erase-block size (commonly 4 MB for eMMC). Pre-allocating one erase block at a
time:

- Ensures the allocated region maps to a natural unit for the FTL's wear-levelling logic.
- Avoids reserving large amounts of space the writer may never fill.
- Makes the allocation step infrequent enough that its overhead is negligible.

### Graceful degradation

`fallocate()` may fail on filesystems that do not support it (e.g., older kernels, tmpfs, network filesystems). The
correct response is to continue writing without pre-allocation; the filesystem will allocate blocks on demand as before.
Log or detect the failure so operators can tell the pre-allocation optimisation is inactive.

---

## Reclaiming Space with ftruncate()

When a file is closed after writing less data than was pre-allocated, `ftruncate(fd, actualSize)` trims the file back to
the actual number of written bytes. Without this step, the pre-allocated zero-filled region remains on disk and wastes
flash capacity.

Always pair `fallocate()` with `ftruncate()` on close to avoid accumulating unused allocated space across many segment
files.

---

## eMMC Flash Characteristics

Understanding the physical constraints of eMMC guides every decision above.

| Property           | Typical value     | Implication                                                     |
| ------------------ | ----------------- | --------------------------------------------------------------- |
| Page size          | 4 KB – 16 KB      | Writes smaller than a page trigger RMW cycles                   |
| Erase block size   | 512 KB – 4 MB     | Only whole erase blocks can be erased; FTL manages mapping      |
| P/E endurance      | 3 000 – 30 000    | Every erase wears the cells; minimise unnecessary erase cycles  |
| Write cache        | Present on device | Writes may sit in device cache until `fsync()` forces commit    |
| FTL wear levelling | Transparent       | Logical addresses do not correspond to fixed physical locations |

### Practical implications

- **Align writes to page size** to avoid RMW on the FTL.
- **Match pre-allocation and segment sizes to erase-block size** so that deleting a segment triggers a clean erase.
- **Minimise `fsync()` frequency** to reduce P/E cycles on the cells that back the dirty pages.
- **Prefer sequential writes** — random small writes fragment the FTL mapping table and reduce GC efficiency.

---

## Segment-Based File Layout

A common pattern for append-only storage is to split the record stream across multiple fixed-size files called
_segments_, rather than growing a single file without bound.

Benefits:

- **Retention by deletion**: deleting old data is a single `unlink()` call — no compaction, no index updates.
- **Bounded write amplification on deletion**: when the segment size equals one erase block, deleting it triggers
  exactly one physical erase operation.
- **Bounded recovery time**: on startup, the system only needs to inspect the last segment to find the valid extent. All
  earlier segments are fully packed and their record counts follow from their file sizes alone.
- **Parallelism-friendly**: a reader can open and read completed segments while the writer works on the newest one.

### Filename-encoded metadata

Embedding the first record number and record size in the filename (e.g., `<uuid>_<firstRecordNumber>_<recordSize>.bin`)
eliminates the need for a separate manifest or index file. Sorting by filename gives the correct chronological order. No
manifest means no risk of the manifest and the data files getting out of sync during a crash.

---

## Self-Describing Recovery Without a Journal

Traditional databases use a write-ahead log (WAL) or journal to recover after a crash. For simple append-only storage, a
sentinel-based approach avoids that overhead entirely.

### How it works

1. Every page written to disk is zero-padded at the tail.
2. Valid records must contain at least one non-zero byte.
3. On startup the reader scans the last page slot by slot; the first all-zero slot marks the end of valid data.
4. All earlier pages are always fully packed — no scanning needed; record counts follow from file sizes.

### What this achieves

- **No separate recovery file**: the data layout encodes its own valid extent.
- **Correct under partial writes**: if a crash occurs mid-page, the partial page on disk either contains the zero
  sentinel naturally (the tail was already zeroed before writing) or the entire page is absent — either way the scan
  finds the correct boundary.
- **Minimal startup cost**: only the final page of the final segment is scanned; everything else is computed.

### Constraint on callers

A record that is entirely zero bytes is indistinguishable from the sentinel. Callers must ensure every valid record
contains at least one non-zero byte. This is typically documented as a contract at the storage interface boundary.

---

## Durability Trade-offs

There is no single right answer for how often to call `fsync()`. The correct choice depends on the data loss tolerance
and the lifetime budget of the flash device.

| Approach                         | Data loss window    | Flash wear impact         |
| -------------------------------- | ------------------- | ------------------------- |
| `fsync()` per record             | Zero                | Very high — avoid         |
| `fsync()` per page (4 KB)        | Up to one page      | High — avoid on eMMC      |
| `fsync()` per segment (4 MB)     | Up to one segment   | One P/E cycle per segment |
| `fsync()` only on clean shutdown | Up to entire stream | Minimal                   |

### Choosing a segment size

With `fsync()` at segment boundaries, the data loss window equals one segment. A 4 MB segment at a moderate write rate
(e.g., a few KB/s) represents many minutes of data. Whether that is acceptable depends on the application. Smaller
segments reduce the data loss window but increase `fsync()` frequency and flash wear.

### Explicit flush as a reader-visibility tool

Calling `fsync()` (or an explicit flush primitive) is sometimes used to make a **partial page** visible to readers, not
primarily for crash durability. Because readers share the OS page cache, a full page is always immediately visible after
`pwrite()` returns. A partial page sitting in an in-memory accumulation buffer is invisible until it is written to disk.
An explicit flush writes and `fsync`-s the partial page, making those records readable to any reader.

This is a different concern from crash durability and the two motivations should not be conflated in the design.

---

## Summary Checklist

- [ ] Use `pwrite()` with page-aligned offsets and sizes (4 KB) to avoid RMW cycles on flash.
- [ ] Do not rely on `pwrite()` for crash durability — only the OS page cache is updated until `fsync()`.
- [ ] Call `fsync()` at bounded checkpoints (e.g., segment boundaries), not per record or per page.
- [ ] Use `fallocate(FALLOC_FL_KEEP_SIZE)` to pre-allocate space in erase-block-sized chunks.
- [ ] Always call `ftruncate()` on segment close to reclaim unused pre-allocated space.
- [ ] Use `sync_file_range(SYNC_FILE_RANGE_WRITE)` to spread writeback load without blocking, but never as a substitute
      for `fsync()`.
- [ ] Embed first-record number and record size in the segment filename to avoid a manifest file.
- [ ] Zero-pad every page tail so a slot-scan can locate the valid extent on startup without a journal.
- [ ] Document the all-zero constraint as a hard contract for callers of the storage interface.
- [ ] Size segments as multiples of the eMMC erase-block size so deletion maps to a single physical erase.
