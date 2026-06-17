# PintOS — Virtual Memory Subsystem

Implementation of a complete Virtual Memory subsystem for the PintOS operating system.

**Course:** Operating Systems | IIT (BHU) Varanasi  
**Authors:** Narsupalli Jayanth Kumar (24075060) · Uyyuru Sudhir (24075090)  
**Guide:** Dr. Bhaskar Biswas  
**Result:** ✅ 109 / 109 tests passing

---

## What's Implemented

| Stage | Feature | Files |
|---|---|---|
| 1 | Supplemental Page Table (SPT) + Stack Growth | `vm/page.h`, `vm/page.c` |
| 2 | Frame Table + Physical Memory Management | `vm/frame.h`, `vm/frame.c` |
| 3 | Page Fault Handling — Demand Paging | `userprog/exception.c` |
| 4 | Swap Management + CLOCK Page Replacement | `vm/swap.h`, `vm/swap.c` |
| 5 | Memory-Mapped Files (`mmap` / `munmap`) | `userprog/syscall.c` |

---

## Architecture Overview

### Supplemental Page Table (SPT)
- Per-process hash table of SPT entries
- Each entry stores: virtual address, page type (file/swap/zero/stack), writable flag, file offset or swap slot, residency state
- Stack growth validated within 32 bytes below `esp` (handles `PUSH`/`PUSHA`)
- Hard 8MB stack size cap enforced

### Frame Table
- Global table tracking all physical frames across all processes
- Per-frame locking — one lock per frame to minimize contention
- `frame_alloc()` triggers CLOCK eviction on `palloc` failure

### CLOCK Page Replacement
- Clock hand sweeps frame table checking hardware accessed bit
- Second-chance: accessed → clear and skip; not accessed → evict
- Dirty file-backed pages written back to file; anonymous pages written to swap; clean file-backed pages discarded
- Disk I/O happens after releasing global frame lock — avoids system-wide stalls

### Swap Manager
- Bitmap-based slot tracking
- Full API: `swap_alloc()`, `swap_read()`, `swap_write()`, `swap_free()`
- All slots released on process exit — no leaks

### Memory-Mapped Files
- Lazy loading — pages brought in through normal fault path on first access
- On `munmap` / process exit: dirty pages flushed, frames released, SPT entries cleaned up
- Per-page locking prevents duplicate loads under concurrent access

---

## Synchronization

| Lock | Scope | Purpose |
|---|---|---|
| `frame_table_lock` | Global | Protects frame list during allocation and eviction scan |
| `frame->lock` | Per-frame | Guards frame metadata; released before disk I/O |
| `spt_entry->lock` | Per-page | Prevents two threads loading the same page simultaneously |
| `swap_bitmap_lock` | Global | Atomic swap slot allocation and release |

---

## Running via Docker

The easiest way to build and test — no local toolchain setup needed.

### Prerequisites
- [Docker](https://docs.docker.com/get-docker/) installed

### Steps

```bash
# Pull the pre-configured PintOS environment
sudo docker run -it --rm johnstarich/pintos bash

# Inside the container — get the source
wget https://github.com/Jayanthk07/PINTOS-VM/archive/refs/heads/master.tar.gz -O master.tar.gz
tar xzf master.tar.gz

# Copy into the PintOS skeleton
cp -r pintos-master/src/* /pintos/

# Build and test VM (Project 3 — this repo)
cd /pintos/vm
make
make check
```

### Other Projects

```bash
cd /pintos/threads && make && make check    # Project 1
cd /pintos/userprog && make && make check   # Project 2
cd /pintos/vm && make && make check         # Project 3 — Virtual Memory
cd /pintos/filesys && make && make check    # Project 4
```

---

## Test Results

```
pass tests/vm/pt-grow-stack
pass tests/vm/pt-grow-stk-sc
pass tests/vm/pt-grow-bad
pass tests/vm/pt-big-stk-obj
pass tests/vm/pt-grow-pusha
...
Result: 109/109 tests passed
```

---

## Repository Structure

```
src/
├── vm/
│   ├── page.h / page.c       # SPT entry struct, fault resolution, mmap tracking
│   ├── frame.h / frame.c     # Frame table, CLOCK eviction, dirty writeback
│   ├── swap.h / swap.c       # Bitmap management, block I/O wrappers
├── userprog/
│   ├── exception.c           # page_fault() — single entry point for all faults
│   └── syscall.c             # mmap/munmap handlers, validation, cleanup
```
