---
title: SFS Lab
description: An AI-assisted SFS solution, with core correctness, optional extensions, and notes on locking.
---

# SFS Lab · Building a Small File System

<p class="article-meta">File systems &amp; concurrency <span class="dot">·</span> Correctness 12/12 <span class="dot">·</span> <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c">sfs-disk.c</a></p>

!!! info "AI-assisted solution"
    This implementation and writeup were completed with AI assistance.

!!! success "Verified locally"
    `make && make test` → **12/12 correctness**, with ThreadSanitizer clean across three fuzzed schedules. Performance is a separate, optional experiment.

SFS is a fixed-layout file system stored in an `mmap`'d disk image. The API is small — open, close, read, write, seek, list, remove, rename — but the implementation must keep three things consistent at once:

1. the **directory namespace** (`name -> file`),
2. the **block graph** on disk, and
3. the **per-descriptor position** each opener sees.

The concurrency part is mostly about ownership: each piece of shared state gets one lock domain.

!!! abstract "The assignment"
    The basic exercise has two steps:

    - **Correctness** — finish the missing operations `sfs_getpos`, `sfs_seek`, and `sfs_rename`, with `rename` required to replace an existing target atomically.
    - **Concurrency** — make the file system thread-safe. Start with one mutex; fine-grained locking is a follow-up optimization.

    The code in this repository is an **extended solution**, based on the handout's `developer` branch. File sizing, zero-block empty files, directory growth, and Unix-style unlink are optional. They are described below to explain the code, not as prerequisites for finishing the lab.

    For the basic starter, use [sfslab on main](https://github.com/LeeXSean/sfslab). The `SFS_Lab/sfslab/` directory here contains completed functions; its adjacent archive contains the extension starter.

## The on-disk model

An SFS image is divided into fixed **512-byte blocks**. Block 0 is the **super block**; every other block starts with a 12-byte header:

``` text
super block (block 0)
+--------------------------------+
| magic[8] + n_blocks[4]         |
| freelist[4] + next_rootdir[4]  |
+--------------------------------+
| 12 bytes unused                |
+--------------------------------+
| 15 x 32-byte root entries      |
+--------------------------------+

ordinary block
+--------------------------------+
| type[4] | prev[4] | next[4]    |
+--------------------------------+
| payload depends on block type  |
+--------------------------------+
```

The block `type` is one of three values read from `sfs-disk.h`:

- `FREE` — block is on the free list,
- `FILE` — block contributes data to one file,
- `DIR` — block stores directory entries.

A `FILE` block spends 12 bytes on its header and leaves **500 bytes** for data:

``` text
FILE block (512 B)
+--------------------------------+
| type[4] | prev[4] | next[4]    |
+--------------------------------+
| 500 B file data                |
+--------------------------------+
```

A `DIR` block uses that same 12-byte header, then 20 bytes of padding, then **15 directory entries** of 32 bytes each:

``` text
DIR block (512 B)
+--------------------------------+
| type[4] | prev[4] | next[4]    |
+--------------------------------+
| 20 B unused                    |
+--------------------------------+
| 15 x 32-byte directory entries |
+--------------------------------+
```

Each directory entry is just:

``` c
typedef struct sfs_dir_entry_t {
    block_id first_block; /**< First block; 0 = unused, EMPTY sentinel = no data */
    uint32_t size;        /**< Size of file in bytes */
    char name[SFS_FILE_NAME_SIZE_LIMIT]; /**< NUL-terminated name */
} sfs_dir_entry_t;
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.h#L90-L95"><code>sfs-disk.h</code> L90–L95</a> (reformatted to fit)</p>

So a file is represented by one directory entry plus a linked chain of `FILE` blocks:

``` text
{name, size, first_block} --> [FILE] <-> [FILE] <-> [FILE] --> 0
```

The root directory begins inside the super block and grows by chaining `DIR` blocks through `next_rootdir` when those embedded 15 entries fill up.

### Optional extension: empty files without data blocks

The optional extension format adds one extra encoding in `sfs-disk.h`:

``` c
/** Developer-branch encoding for a live empty file with no data block.
    Valid block IDs are always smaller than the uint32_t block count, so this
    value can never name a mapped block. */
#define SFS_EMPTY_FILE_BLOCK UINT32_MAX
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.h#L49-L52"><code>sfs-disk.h</code> L49–L52</a> (reformatted to fit)</p>

That creates a three-way distinction for `first_block`:

| `first_block` value | Meaning |
|---|---|
| `0` | unused directory slot |
| `SFS_EMPTY_FILE_BLOCK` | live empty file |
| any other block id | first data block of a live file |

This keeps empty files live without burning a 512-byte data block, while still making occupied directory slots unambiguous.

---

## One open file becomes three objects

The most important design choice is not on disk at all. SFS separates **the name in the directory**, **the shared state of one file**, and **the per-descriptor position**:

``` text
openFileDescTable[fd]
        |
        v
  sfs_mem_filedesc_t          one per open descriptor
  +------------------+
  | currPos          |
  | fileEntry -------+---+
  +------------------+   |
                         v
                   sfs_mem_file_t       one per open file
                   +----------------+
                   | refCount       |
                   | unlinked       |
                   | pthread lock   |
                   | diskFile ------+---+
                   +----------------+   |
                                        v
                                  sfs_dir_entry_t
                                  { name, size, first_block }
```

The two in-memory structs are:

``` c
/** This struct corresponds to what CS:APP calls a "v-node table" entry. */
typedef struct sfs_mem_file_t {
    uint32_t refCount; int tableIndex; int unlinked; pthread_mutex_t lock;
    sfs_dir_entry_t *diskFile; sfs_dir_entry_t unlinkedFile;
} sfs_mem_file_t;

/** This struct corresponds to what CS:APP calls an "open file table" entry.
    The "descriptor table" is the openFileDescTable array itself. */
typedef struct sfs_mem_filedesc_t {
    sfs_mem_file_t *fileEntry; size_t currPos;
} sfs_mem_filedesc_t;
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c#L70-L87"><code>sfs-disk.c</code> L70–L87</a> (reformatted to fit)</p>

This split gives two immediate properties:

- Opening the same file twice gives two descriptors with **independent** `currPos` values.
- Every opener still shares one `sfs_mem_file_t`, so file size, block chain, and unlink state have one mutex and one reference count.

### Optional extension: Unix-style unlink

The helper `unlinkDirectoryEntry` is the core of both `sfs_remove` and overwrite-style `sfs_rename`:

``` c
/** Remove ENTRY while the directory and open-file tables are write-locked. */
static void unlinkDirectoryEntry(sfs_dir_entry_t *entry) {
    sfs_mem_file_t *open = findOpenFile(entry);
    if (open != NULL) {
        pthread_mutex_lock(&open->lock);
        open->unlinkedFile = *entry;
        open->diskFile = &open->unlinkedFile;
        open->unlinked = 1;
        pthread_mutex_unlock(&open->lock);
    } else if (entry->first_block != SFS_EMPTY_FILE_BLOCK) {
        freeBlocks(entry->first_block);
    }
    memset(entry, 0, sizeof *entry);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c#L486-L503"><code>sfs-disk.c</code> L486–L503</a> (reformatted to fit)</p>

If the file is closed, its blocks can go back to the free list immediately. If it is open, the directory entry is copied into `unlinkedFile`, `diskFile` is redirected to that private copy, and the visible directory slot is cleared.

That gives SFS the Unix rule in compact form: removing a file destroys its **name** now, but destroys its **storage** only after the last open descriptor disappears.

`sfs_close` finishes the job. When `refCount` drops to zero, it checks `unlinked` and frees the saved block chain.

---

## Part 1 · Completing the API

### `getpos` and `seek` are descriptor-local

`sfs_getpos` is the simple one: validate `fd`, return `currPos`, unlock the descriptor slot.

`sfs_seek` is slightly more careful because it has to move backward without invoking signed-overflow undefined behavior:

``` c
ssize_t sfs_seek(int fd, ssize_t delta) {
    sfs_mem_filedesc_t *descriptor = lockDescriptor(fd);
    if (descriptor == NULL)
        return -EBADF;
    sfs_mem_file_t *file = descriptor->fileEntry;
    pthread_mutex_lock(&file->lock);
    size_t position = descriptor->currPos;
    size_t file_size = file->diskFile->size;
    if (delta < 0) {
        size_t distance = (size_t)(-(delta + 1)) + 1;
        position = distance > position ? 0 : position - distance;
    } else {
        size_t distance = (size_t)delta;
        position =
            distance > file_size - position ? file_size : position + distance;
    }
    descriptor->currPos = position;
    pthread_mutex_unlock(&file->lock); pthread_mutex_unlock(&descriptorLocks[fd]);
    return (ssize_t)position;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c#L928-L952"><code>sfs-disk.c</code> L928–L952</a> (reformatted to fit)</p>

The rule matches the API comment in `sfs-api.h`: the result is always clamped into `[0, file_size]`. SFS deliberately does **not** allow a seek position past EOF.

The `-(delta + 1) + 1` pattern matters because directly negating the most negative `ssize_t` value is undefined.

### `rename` is one namespace transaction

The required behavior is stronger than “change the string in the directory entry.” If `new_name` already exists, SFS has to replace it **atomically**: no other thread should ever observe a gap where `new_name` does not exist.

The implementation takes both write locks — the directory and the open-file table — after validating its arguments, and holds them across the whole namespace change:

Branch order: validate the names · take both write locks · `-ENOENT` if the source is missing · a same-name rename is a no-op · unlink an existing target · rewrite the name bytes · unlock in reverse order.

``` c
int sfs_rename(const char *old_name, const char *new_name) {
    if (old_name[0] == '\0' || new_name[0] == '\0')
        return -EINVAL;

    if (strnlen(old_name, SFS_FILE_NAME_SIZE_LIMIT + 1) + 1 >
            SFS_FILE_NAME_SIZE_LIMIT ||
        strnlen(new_name, SFS_FILE_NAME_SIZE_LIMIT + 1) + 1 >
            SFS_FILE_NAME_SIZE_LIMIT)
        return -ENAMETOOLONG;

    if (getSFSStatus() < 0)
        return -ENOMEDIUM;

    pthread_rwlock_wrlock(&directoryLock); pthread_rwlock_wrlock(&openTableLock);
    sfs_dir_entry_t *old_entry = findDirectoryEntry(old_name, NULL, NULL);
    if (old_entry == NULL) {
        pthread_rwlock_unlock(&openTableLock); pthread_rwlock_unlock(&directoryLock);
        return -ENOENT;
    }

    if (strcmp(old_name, new_name) == 0) {
        pthread_rwlock_unlock(&openTableLock); pthread_rwlock_unlock(&directoryLock);
        return 0;
    }

    sfs_dir_entry_t *new_entry = findDirectoryEntry(new_name, NULL, NULL);
    if (new_entry != NULL)
        unlinkDirectoryEntry(new_entry);

    size_t len = strlen(new_name);
    memcpy(old_entry->name, new_name, len);
    memset(old_entry->name + len, '\0', SFS_FILE_NAME_SIZE_LIMIT - len);

    pthread_rwlock_unlock(&openTableLock); pthread_rwlock_unlock(&directoryLock);
    return 0;
}
```
<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c#L985-L1027"><code>sfs-disk.c</code> L985–L1027</a> (reformatted to fit)</p>

`unlinkDirectoryEntry(new_entry)` and the rename of `old_entry` happen under the same lock interval. Other threads see either the old namespace or the finished new one, never the half-updated state in between.

Because overwrite-rename uses the same unlink path as `sfs_remove`, an already-open target file keeps working after replacement. Its descriptors now point at `unlinkedFile`, and its blocks are reclaimed only on final close.

### Writing grows the file atomically on allocation failure

The graded API permits short writes, but this implementation takes an all-or-nothing path when a write needs more blocks: if the final size cannot be allocated, it returns `-ENOSPC` before touching the file.

The whole operation is decided before a byte is copied: every additional block is reserved up front, and a failure there returns before the file is touched.

Branch order: lock the descriptor and file · reject `-EFBIG` and zero-length writes · reserve the blocks a growth needs · copy 500-byte chunks · attach the reserved chain only at the old tail · unlock.

``` c
ssize_t sfs_write(int fd, const char *buf, size_t len) {
    sfs_mem_filedesc_t *tFile = lockDescriptor(fd);
    if (tFile == NULL)
        return -EBADF;
    sfs_mem_file_t *file = tFile->fileEntry;
    pthread_mutex_lock(&file->lock);

    size_t fileSize = file->diskFile->size;
    size_t currPos = tFile->currPos;
    assert(currPos <= fileSize);

    // This implementation does not do a partial write if there is
    // insufficient space on disk for the complete write; it always
    // either writes all 'len' bytes, or none.
    if (len > SFS_MAX_FILE_SIZE - currPos) {
        pthread_mutex_unlock(&file->lock); pthread_mutex_unlock(&descriptorLocks[fd]);
        return -EFBIG;
    }
    if (len == 0) {
        pthread_mutex_unlock(&file->lock); pthread_mutex_unlock(&descriptorLocks[fd]);
        return 0;
    }

    size_t fileAllocSize =
        (size_t)allocatedBlocksForFile(file->diskFile) * BLOCK_DATA_SIZE;
    size_t endPos = len + currPos;
    size_t toWrite = len;

    // If we need to enlarge the file, do so now, and if we can't make
    // it big enough, fail the whole operation.
    block_id firstNewId = 0;
    if (endPos > fileAllocSize) {
        size_t fileNewAllocSize = roundUp(endPos, BLOCK_DATA_SIZE);
        uint32_t addlBlocks =
            (uint32_t)((fileNewAllocSize - fileAllocSize) / BLOCK_DATA_SIZE);
        assert(addlBlocks >= 1);

        firstNewId = allocateBlocks(addlBlocks, SFS_BLOCK_TYPE_FILE);
        if (firstNewId == 0) {
            pthread_mutex_unlock(&file->lock);
            pthread_mutex_unlock(&descriptorLocks[fd]); return -ENOSPC;
        }
    }

    // Copy chunks of data from the caller's buffer to the mapped disk image.
    // See comments above the very similar loop in sfs_read() for more detail.
    sfs_block_file_t *diskBlock;
    block_id first = file->diskFile->first_block;
    if (first == SFS_EMPTY_FILE_BLOCK) {
        assert(fileSize == 0 && currPos == 0 && firstNewId != 0);
        diskBlock = accessFileBlock(firstNewId);
        file->diskFile->first_block = firstNewId;
        firstNewId = 0;
    } else {
        assert(first != 0);
        diskBlock = accessFileBlock(blockForPosition(first, currPos));
    }
    size_t blockPos = currPos % BLOCK_DATA_SIZE;
    size_t chunkSize =
        sizeMin(roundUp(currPos, BLOCK_DATA_SIZE) - currPos, toWrite);
    for (;;) {
        // The chunk size can be zero on the first iteration, if the
        // starting position was exactly at a block boundary.
        if (chunkSize > 0) {
            memcpy(&diskBlock->data[blockPos], buf, chunkSize);
            buf += chunkSize;
            toWrite -= chunkSize;
        }
        if (toWrite == 0)
            break;

        blockPos = 0;
        chunkSize = sizeMin(BLOCK_DATA_SIZE, toWrite);
        sfs_block_file_t *nextBlock = accessFileBlock(diskBlock->h.next_block);
        if (nextBlock == NULL) {
            // We should only get here once, at most, per write call.
            assert(firstNewId != 0);
            // We have just advanced the file position to the end of the
            // original allocation for the file.  Attach the additional
            // blocks beginning at 'firstNewId' to the end of the file,
            // and continue.
            nextBlock = accessFileBlock(firstNewId);
            diskBlock->h.next_block = firstNewId;
            nextBlock->h.prev_block = idOfBlock(&diskBlock->h);
            firstNewId = 0;
        }
        diskBlock = nextBlock;
    }

    tFile->currPos = endPos;
    if (endPos > fileSize) {
        assert(endPos <= SFS_MAX_FILE_SIZE);
        file->diskFile->size = (uint32_t)endPos;
    }
    pthread_mutex_unlock(&file->lock); pthread_mutex_unlock(&descriptorLocks[fd]);
    return (ssize_t)len;
}
```
<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c#L681-L793"><code>sfs-disk.c</code> L681–L793</a> (reformatted to fit)</p>

Only after that reservation succeeds does the copy loop run. If the file used to be empty, the first successful write swaps the empty-file sentinel for the new chain head.

If the file already had data, the new chain is attached when the copy loop reaches the old tail.

So allocation failure leaves the old file untouched, and successful growth preserves a valid block chain throughout.

### Reads and writes advance in 500-byte chunks

Once the current block is known, both `sfs_read` and `sfs_write` iterate one chunk at a time, never crossing a block boundary in a single copy:

Branch order: lock · clamp the request to what the file holds · copy chunk by chunk · unlock.

``` c
ssize_t sfs_read(int fd, char *buf, size_t len) {
    sfs_mem_filedesc_t *tFile = lockDescriptor(fd);
    if (tFile == NULL)
        return -EBADF;
    sfs_mem_file_t *file = tFile->fileEntry;
    pthread_mutex_lock(&file->lock);

    // We are going to read 'len' bytes, or the amount of data remaining
    // in the file, whichever is smaller.
    // This subtraction cannot produce a value larger than SSIZE_MAX
    // because it's impossible for a file in SFS to be that large.
    size_t fileSize = file->diskFile->size;
    size_t currPos = tFile->currPos;

    assert(currPos <= fileSize);
    size_t totalToRead = sizeMin(fileSize - currPos, len);

    size_t toRead = totalToRead;
    if (toRead == 0) {
        pthread_mutex_unlock(&file->lock); pthread_mutex_unlock(&descriptorLocks[fd]);
        return 0;
    }

    // Copy chunks of data from the mapped disk image to the caller's buffer.
    //
    // Each chunk is the smaller of:
    //  - the amount of data still to be read
    //  - the amount of data between currPos and the end of the current block
    // This number can be different from BLOCK_DATA_SIZE only for the
    // very first and the very last chunk of a read operation.
    //
    // Each chunk starts at the beginning of a disk block's data area,
    // except the very first chunk, which will begin in the middle of a
    // data area if the previous read or seek operation left the file
    // position not a multiple of BLOCK_DATA_SIZE.
    block_id first = file->diskFile->first_block;
    assert(first != 0 && first != SFS_EMPTY_FILE_BLOCK);
    sfs_block_file_t *diskBlock =
        accessFileBlock(blockForPosition(first, currPos));
    size_t blockPos = currPos % BLOCK_DATA_SIZE;
    size_t chunkSize =
        sizeMin(roundUp(currPos, BLOCK_DATA_SIZE) - currPos, toRead);
    for (;;) {
        // The chunk size can be zero on the first iteration, if the
        // starting position was exactly at a block boundary.
        if (chunkSize > 0) {
            memcpy(buf, &diskBlock->data[blockPos], chunkSize);
            buf += chunkSize;
            toRead -= chunkSize;
        }
        if (toRead == 0)
            break;

        blockPos = 0;
        chunkSize = sizeMin(BLOCK_DATA_SIZE, toRead);
        diskBlock = accessFileBlock(diskBlock->h.next_block);
        // This could only happen legitimately if we were reading to the end
        // of a file whose size was an exact multiple of BLOCK_DATA_SIZE, but
        // then we would already have exited the loop.
        assert(diskBlock != NULL);
    }

    tFile->currPos = currPos + totalToRead;

    pthread_mutex_unlock(&file->lock); pthread_mutex_unlock(&descriptorLocks[fd]);
    return (ssize_t)totalToRead;
}
```
<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c#L607-L679"><code>sfs-disk.c</code> L607–L679</a> (reformatted to fit)</p>

At a nonzero multiple of 500 the first `chunkSize` is zero, so the loop advances once before copying; at position 0 `roundUp` returns a full block instead, and every other position consumes just the remainder of the current block. That keeps the logic uniform for boundary and non-boundary positions alike.

---

## Part 2 · Optional refinement: finer-grained locks

The code uses five lock domains:

| Lock | Protects |
|---|---|
| `directoryLock` (`rwlock`) | directory slots, root-directory growth, namespace changes |
| `openTableLock` (`rwlock`) | table membership and lifetime, `refCount`, `unlinked`, table scans |
| `descriptorLocks[fd]` (`mutex`) | one descriptor slot and its `currPos` |
| `fileEntry->lock` (`mutex`) | `diskFile` target, one file's size, block chain, and contents |
| `allocationLock` (`mutex`) | free-list allocation and reclamation |

The lock order is fixed:

``` text
directory -> open table -> descriptor slot -> file -> allocator
```

Not every function uses every lock, but no function reverses that order.

### The hot path avoids the open-file table

The scalability pivot is `lockDescriptor(fd)`:

``` c
/** Lock FD's permanent table slot and return its live descriptor, if any. */
static sfs_mem_filedesc_t *lockDescriptor(int fd) {
    if (fd < 0 || fd >= OPEN_FILE_LIMIT)
        return NULL;
    pthread_once(&descriptorLocksOnce, initializeDescriptorLocks);
    pthread_mutex_lock(&descriptorLocks[fd]);
    sfs_mem_filedesc_t *descriptor = openFileDescTable[fd];
    if (descriptor == NULL)
        pthread_mutex_unlock(&descriptorLocks[fd]);
    return descriptor;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c#L114-L125"><code>sfs-disk.c</code> L114–L125</a> (reformatted to fit)</p>

A `getpos` call needs only its descriptor slot; operations that inspect or change file state add the per-file lock:

``` text
getpos:              fd slot
read / write / seek: fd slot -> file
```

and a write that grows the file adds the allocator lock at the end:

``` text
fd slot -> file -> allocator
```

That means two threads operating on different files do **not** need the directory lock, and they do **not** need the open-table lock either. They contend only if they actually share a descriptor, a file, or the free list.

The open-table lock remains necessary for `open`, `close`, `remove`, `rename`, and `ftruncate`, because those paths create or destroy global relationships. But it is no longer in the middle of millions of timed I/O calls.

### Descriptor positions are simple; block locations are derived

The starter design this article grew from cached more block-related state in each descriptor. This implementation keeps only `currPos` — the two-field `sfs_mem_filedesc_t` shown [above](#one-open-file-becomes-three-objects) — and derives the current block whenever it is needed, from the file's head block and the byte position:

``` c
static block_id blockForPosition(block_id first, size_t position) {
    if (first == 0)
        return 0;
    uint32_t index = (uint32_t)(position / BLOCK_DATA_SIZE);
    if (position != 0 && position % BLOCK_DATA_SIZE == 0)
        index--;
    return idOfBlock(&fileBlockAt(first, index)->h);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c#L287-L295"><code>sfs-disk.c</code> L287–L295</a> (reformatted to fit)</p>

That removes one class of synchronization work. When an empty file receives its first block, or `ftruncate` frees tail blocks, there are no cached block IDs scattered across live descriptors that now need repair. The only persistent per-descriptor state is the byte position.

Locating a block requires walking from the file head. This is simple for small files but repeated I/O near the end of a large file can become expensive; caching block positions would be a separate optimization.

### Directory readers sometimes need the file lock too

Directory operations cannot reason only about names.

`findDirectoryEntry` and `sfs_list` both do this before deciding whether a slot is occupied:

Scan order: the 15 entries in the super block · every chained `DIR` block after that — under the file lock when a slot is live.

``` c
static sfs_dir_entry_t *findDirectoryEntry(const char *name,
                                           sfs_dir_entry_t **empty_out,
                                           block_id *last_dir_out) {
    sfs_filesystem_t *super = accessSuperBlock();
    sfs_dir_entry_t *empty = NULL;
    for (size_t i = 0; i < DIR_ENTRIES_PER_BLOCK; i++) {
        sfs_dir_entry_t *entry = &super->files[i];
        sfs_mem_file_t *open = findOpenFile(entry);
        if (open != NULL)
            pthread_mutex_lock(&open->lock);
        int occupied = entry->first_block != 0;
        int matches = occupied && strcmp(entry->name, name) == 0;
        if (open != NULL)
            pthread_mutex_unlock(&open->lock);
        if (matches)
            return entry;
        if (empty == NULL && !occupied)
            empty = entry;
    }

    block_id last = 0;
    for (block_id id = super->next_rootdir; id != 0;) {
        sfs_block_dir_t *dir = accessDirectoryBlock(id);
        last = id;
        for (size_t i = 0; i < DIR_ENTRIES_PER_BLOCK; i++) {
            sfs_dir_entry_t *entry = &dir->files[i];
            sfs_mem_file_t *open = findOpenFile(entry);
            if (open != NULL)
                pthread_mutex_lock(&open->lock);
            int occupied = entry->first_block != 0;
            int matches = occupied && strcmp(entry->name, name) == 0;
            if (open != NULL)
                pthread_mutex_unlock(&open->lock);
            if (matches)
                return entry;
            if (empty == NULL && !occupied)
                empty = entry;
        }
        id = dir->h.next_block;
    }
    if (empty_out != NULL)
        *empty_out = empty;
    if (last_dir_out != NULL)
        *last_dir_out = last;
    return NULL;
}
```
<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/SFS_Lab/sfslab/sfs-disk.c#L402-L451"><code>sfs-disk.c</code> L402–L451</a> (reformatted to fit)</p>

Why? Because `first_block` lives inside the directory entry, but it changes under the file lock when:

- an empty file receives its first data block,
- `ftruncate` grows or shrinks the allocation.

So a directory scan sometimes depends on file metadata that is mutable elsewhere. The scan therefore follows that data dependency to the lock that already owns it.

### `sfs_list` uses a physical slot cookie

The listing API in `sfs-api.h` returns one name per call through an opaque cookie. In this implementation the cookie is just the next physical directory slot, encoded as a `void *` and counted across:

1. the 15 root entries inside the super block, then
2. every chained `DIR` block after that.

That is why `sfs_list` can resume exactly where it left off without materializing a separate iterator object. The API still requires callers to iterate until a nonzero status, but this implementation acquires and releases both read locks within each call; the cookie stores only the slot position.

---

## Verification

Run from `SFS_Lab/sfslab/`:

```sh
make
make test
make developer-test x-traces
```

Verified on September 5, 2026, using the updated local driver:

| Check | Result |
|---|---|
| Category A · feature tests | 5/5 |
| Category B · sequential correctness | 4/4 |
| Category C · actual concurrent calls | 3/3 |
| ThreadSanitizer fuzzed schedules | 3 clean runs |
| Correctness | 12/12 |
| Optional file-size trace | 1/1 |
| Optional directory/unlink/empty-file traces | 3/3 |

Unlike the old driver, the normal C traces no longer serialize calls on behalf
of the implementation. A passing result tests the locks in the solution itself.

To explore performance separately:

```sh
make grade
# Or, after calibration:
./test-sfs --benchmark
```

The optional 22-point scale is local feedback, not an official CMU grade.
Throughput depends on the machine and workload; the earlier 22/22 result is
not the completion criterion for the current basic exercise.
