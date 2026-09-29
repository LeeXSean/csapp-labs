---
title: Malloc Lab
description: A dynamic memory allocator from scratch — six designs, from one implicit list to twelve segregated rings.
---

# Malloc Lab · Writing malloc by Hand

<p class="article-meta">Dynamic memory <span class="dot">·</span> Score 97/100 <span class="dot">·</span> <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c">mm.c</a></p>

!!! success "Verified locally"
    `./mdriver -V` → **97 / 100** (util 57 + throughput 40), averaging 95% utilization over 11 traces.

The course's toughest assignment: implement `mm_malloc`, `mm_free`, and `mm_realloc` from scratch on a raw heap from `mem_sbrk`. Every byte of metadata and every wasted gap is counted.

!!! abstract "The assignment"
    You manage a single contiguous heap that only grows (via `mem_sbrk`). You must provide:

    - `mm_init` — set up the empty heap,
    - `mm_malloc(size)` — return an 8-byte-aligned block with room for `size` payload bytes,
    - `mm_free(ptr)` — release a block,
    - `mm_realloc(ptr, size)` — resize while preserving the old contents.

    The grader scores two quantities that pull in **opposite directions**:

    - **Utilization** — payload bytes actually requested ÷ peak heap size. Rewards *small* metadata and *tight* packing.
    - **Throughput** — operations per second. Rewards *fast* searches and *cheap* bookkeeping.

    Push utilization and you add splitting, coalescing, and search work that hurts throughput. Push throughput and you leak internal fragmentation. The design is about balancing the two.

## From one list to twelve { data-toc-label="Evolution" }

This allocator is the sixth shape the code took, not the first. Each redesign fixed the pain the previous one exposed, and each left traces in the final source. The route:

**1 · Implicit free list.** Every block carries a header and footer, and the fit search walks *every block in the heap*, allocated or not. Hopeless on throughput: on allocation-heavy traces the search walks thousands of blocks that were never candidates.

``` text
   implicit -- the fit search visits EVERY block, allocated or not:

     [A][A][F][A][F][A][A][F][A][F]      one list = the whole heap

   explicit -- free blocks thread a list through their own payloads:

     [F] <-> [F] <-> [F]                 allocated blocks never visited
```

**2 · Explicit, doubly linked.** Free blocks recycle their first two payload words as `prev`/`next` links, so the search touches only genuine candidates and `free` becomes an O(1) LIFO insert. The price moves to the edges: with `NULL` at both ends, insert and remove split into empty-list, head, middle, and tail cases: four ways to get the two pointers wrong.

**3 · Close the ring.** Make the list circular: the last block links back to the first. Ends stop existing, so end-handling stops existing — a removal in the middle is the same splice as anywhere else. Two special cases survive: the empty list, and removing the very block the head pointer names.

``` text
   NULL-ended    NULL <- [F] <-> [F] <-> [F] -> NULL

   circular         .--> [F] <-> [F] <-> [F] <--.
                    '---------------------------'

   sentinel         .--> [S] <-> [F] <-> [F] <--.
                    '---------------------------'
```

**4 · Plant a sentinel.** The classic finisher: a permanent dummy node `[S]` that always sits in the ring. "Empty" becomes "the sentinel alone," the head never moves, and insert/remove collapse into two unconditional pointer splices — no branches at either end.

**5 · Segregate by size.** One list still fits badly as the heap fills: either skip past piles of too-small blocks, or take the first oversized one and lose utilization. Splitting free blocks into **12 geometric size classes** shrinks every search to blocks that could plausibly fit. Under the sentinel pattern, that means twelve rings — and twelve sentinels.

**6 · Fire the sentinels, keep the rings.** Two constraints work against the dummies. Malloclab allows only *scalar* globals — no global arrays or structs — so the whole apparatus must live inside the heap, where every byte counts against utilization. And a sentinel needs two words (`prev` + `next`) where a bare head pointer needs one: `12 × 8 = 96` bytes against `12 × 4 = 48`. So the final code `mem_sbrk`s a 48-byte array of one-word class heads as the heap's very first bytes, and the dummies go. The branches the sentinels had erased walk right back in — the empty-class check in `insert_node`, the head hand-off in `remove_node` — two predictable compares, bought for half the metadata. The rings themselves stayed, so the circular links in today's code are the leftover of that sentinel era.

## Design at a glance

| Decision | Choice here | Buys us |
|----------|-------------|---------|
| Free-block organization | **Segregated** explicit free lists, 12 classes | Search only size-relevant free blocks, never allocated ones |
| Allocated-block metadata | **Footer-less** (header only) | 4 fewer overhead bytes per live block |
| Free-list links | One 32-bit word each, in the free payload | 16-byte minimum free block |
| Coalescing | **Immediate**, on every free | No deferred-fragmentation bookkeeping |
| Fit policy | First class containing a fit, **best fit within it** | Tighter than first-fit, with class-local scans |
| Realloc | Reuse adjacent space before allocating a replacement | Avoids most malloc-copy-free cycles |

The constants that anchor everything:

``` c
#define WSIZE               4           /* Word and header/footer size (bytes) */
#define DSIZE               8           /* Double word size (bytes) */
#define CHUNKSIZE           (1<<6)     /* Extend Heap by this amount (bytes) */
#define CLASSNUM            12
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L44-L47"><code>mm.c</code> L44–L47</a> (reformatted to fit)</p>

---

## Block anatomy

Every block starts with a one-word (4-byte) **header**. Because sizes are always a multiple of 8, the low three bits are free real estate for flags — here two of them are used:

``` text
 bit:  31 ........................ 3   2    1    0
      +-----------------------------+----+----+----+
      |          block size         | 0  | PA | A  |
      +-----------------------------+----+----+----+
        A  (bit 0) = is THIS block allocated?
        PA (bit 1) = is the PREVIOUS block allocated?
```

That `PA` (previous-allocated) bit carries the whole design — more on it in a moment. The header is decoded with three tiny macros:

``` c
/* Read the size and allocation fields from address p */
#define GET_SIZE(p)         (GET(p) & ~0x7)      // (1)
#define GET_ALLOC(p)        (GET(p) & 0x1)       // (2)
#define GET_PREV_ALLOC(p)   (GET(p) & 0x2)       // (3)
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L62-L65"><code>mm.c</code> L62–L65</a> (reformatted to fit)</p>

1.  Mask off the three flag bits to recover the size.
2.  Bit 0: is *this* block allocated?
3.  Bit 1: is the *previous* (adjacent, lower-address) block allocated? Reading this is what lets us skip the previous block's footer.

An **allocated** block carries nothing but the header and the payload. A **free** block is richer: it reuses the first two payload words as list links, and keeps a **footer** (a copy of the header) at its tail:

``` text
   Allocated block                 Free block (not to scale)
  +------------------+  <- header  +------------------+  <- header
  |  size | PA | 1   |             |  size | PA | 0   |
  +------------------+  <- bp      +------------------+  <- bp
  |                  |             |  prev free ptr   |  (4 bytes)
  |     payload      |             +------------------+
  |                  |             |  next free ptr   |  (4 bytes)
  |                  |             +------------------+
  |                  |             |  free payload*   |
  +------------------+             +------------------+
       no footer!                  |  size | PA | 0   |  <- footer
                                   +------------------+

   * absent from a minimum 16-byte free block
```

!!! note "Why the links are one word"
    A free block must hold a header + two links + a footer, so link width sets the minimum block size: `4 + 4 + 4 + 4 = 16` bytes. The driver builds with `-m32`, where a pointer is naturally 4 bytes — and the link macros pin that layout down explicitly by storing every link as an `unsigned int`:

    ``` c
    /* Read and write free-list pointers stored inside free block payloads */
    #define GET_PTR(p)          ((char *)(uintptr_t)GET(p))
    #define PUT_PTR(p, val)     (PUT((p), (unsigned int)(uintptr_t)(val)))
    ```

    Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L58-L60"><code>mm.c</code> L58–L60</a>
    {: .code-source }

    Under `-m32` the casts are a no-op; on a 64-bit rebuild they become genuine truncation, which stays safe only while every heap address fits in 32 bits. Either way the layout — and the 16-byte minimum — holds, where native 8-byte links would push it to 24.

Navigation between neighbors is pure pointer arithmetic off the size in the header (and, for the *previous* block, its footer):

``` c
/* Given block ptr bp, compute address of its header and footer */
#define HDRP(bp)            ((char *)(bp) - WSIZE)
#define FTRP(bp)            ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)

/* Given block ptr bp, compute address of next and previous blocks */
#define NEXT_BLKP(bp)       ((char *)(bp) + GET_SIZE((char *)(bp) - WSIZE))
#define PREV_BLKP(bp)       ((char *)(bp) - GET_SIZE((char *)(bp) - DSIZE))   // (1)
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L71-L77"><code>mm.c</code> L71–L77</a> (reformatted to fit)</p>

1.  `PREV_BLKP` reads the **previous block's footer** (`bp - DSIZE`) to learn its size. This only works if the previous block *has* a footer — i.e. if it is free. That's exactly the case in which we need it, which is what makes the next optimization safe.

## Footer-less allocated blocks { data-toc-label="Footer-less blocks" }

The classic CS:APP allocator gives every block a footer so that, when freeing block *B*, we can look at *B*'s left neighbor's footer to decide whether to coalesce backward. Footers on **allocated** blocks are pure overhead, though — 4 bytes lost on every live object.

The `PA` bit. Each block's header already records whether its **left neighbor** is allocated. So:

- To coalesce *backward*, check `GET_PREV_ALLOC` — no footer read needed.
- Only when the neighbor turns out to be **free** do we consult its footer (via `PREV_BLKP`), and free blocks still carry one.

The result: **allocated blocks drop their footer entirely** (4-byte overhead → header only), while the backward-coalescing machinery keeps working. The one obligation is discipline: every path that allocates or frees a block must update the `PA` bit of the *following* block. For example, `mm_free` clears the next block's `PA` bit:

``` c
void mm_free(void *bp) {
    size_t size;

    if (bp == NULL)
        return;

    size = GET_SIZE(HDRP(bp));
    PUT(HDRP(bp), PACK(size, GET_PREV_ALLOC(HDRP(bp))));
    PUT(FTRP(bp), PACK(size, GET_PREV_ALLOC(HDRP(bp))));
    PUT(HDRP(NEXT_BLKP(bp)), PACK(GET_SIZE(HDRP(NEXT_BLKP(bp))),
            GET_ALLOC(HDRP(NEXT_BLKP(bp)))));   // (1)

    coalesce(bp);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L371-L384"><code>mm.c</code> L371–L384</a> (reformatted to fit)</p>

1.  Rewrites the next header with its own size and alloc bit, but *without* the `0x2` flag — signalling "the block before me is now free."

## Segregated free lists

Free blocks are bucketed into **12 size classes**, each an independent circular doubly-linked list. The 12 head pointers live in a small array carved from the very front of the heap — they can't be a C global, because the lab allows only scalar globals:

``` c
int mm_init(void) {
    char *bp; int i;

    if ((seg_free_lists = mem_sbrk(CLASSNUM * WSIZE)) == (void *)-1)
        return -1;
    for (i = 0; i < CLASSNUM; i++) { SET_SEG_HEAD(i, NULL); }

    /* Create the initial empty heap */
    if ((heap_listp = mem_sbrk(4*WSIZE)) == (void *)-1)
        return -1;
    PUT(heap_listp, 0);                             /* Alignment padding */
    PUT(heap_listp + (1*WSIZE), PACK(DSIZE, 1));    /* Prologue header */
    PUT(heap_listp + (2*WSIZE), PACK(DSIZE, 1));    /* Prologue footer */
    PUT(heap_listp + (3*WSIZE), PACK(0, 0x3));      /* Epilogue header */
    heap_listp += (2*WSIZE);

    /* Extend the empty heap with a free block of CHUNKSIZE bytes */
    if ((bp = extend_heap(CHUNKSIZE/WSIZE)) == NULL)
        return -1;
    return 0;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L306-L330"><code>mm.c</code> L306–L330</a> (reformatted to fit)</p>

``` text
 seg_free_lists (12 heads)
 +----------+
 | class 0  | --> NULL
 | class 1  | --> F1  (ring shown below)
 | class 2  | --> NULL
 |   ...    |
 | class 11 | --> NULL
 +----------+

 class 1 ring:
          F3.next -> F1
       .-----------------.
       v                 |
      [F1] <-> [F2] <-> [F3]
       |                 ^
       '-----------------'
          F1.prev -> F3
```

A request's class is chosen from its size. The index grows **geometrically** — each class covers roughly double the size range of the one before, and class 11 is a catch-all:

``` c
static size_t find_index(size_t asize) {
    size_t index;
    size_t count = 0;

    if (asize <= 2*DSIZE)                     // (1)
        return 0;

    index = (asize-WSIZE-1) >> 4;             // (2)
    while (index && count < (CLASSNUM-1)) {       // (3)
        index >>= 1;
        count += 1;
    }
    index = count;

    return index;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L119-L135"><code>mm.c</code> L119–L135</a> (reformatted to fit)</p>

1.  The smallest blocks (16 bytes) all land in class 0.
2.  Scale the size down into a coarse bucket number.
3.  Count how many halvings reduce it to zero — i.e. `⌊log2⌋` — capped at 11. Bigger blocks climb to higher classes.

| Class | Holds blocks of (bytes) |
|-------|-------------------------|
| 0 | 16 |
| 1 | 24 – 32 |
| 2 | 40 – 64 |
| 3 | 72 – 128 |
| ... | roughly doubling each step |
| 11 | everything larger (catch-all) |

### Rounding payloads for reuse { data-toc-label="Rounding payloads" }

Before sizing a block, small and medium **payloads** are rounded up to the next power of two (between 16 and 512). Two allocations that round to the same class become perfectly interchangeable later, which cuts fragmentation over a long run of malloc/free:

``` c
static size_t round_payload(size_t size) {
    size_t rounded = 16;

    if (size <= 16)
        return size;

    while (rounded < size && rounded < 512)
        rounded <<= 1;

    if (rounded >= size && rounded <= 512)
        return rounded;

    return size;                    // (1)
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L96-L110"><code>mm.c</code> L96–L110</a> (reformatted to fit)</p>

1.  Very large requests (> 512) are left exactly as asked — rounding them would waste too much.

### Insert, remove, and the fit search { data-toc-label="Insert / remove / fit" }

Insertion is **LIFO** at the head of the class list (the just-freed block is the likeliest to still be warm in cache). The empty class is the only special case: a single node is its own predecessor and successor, so the block is linked to itself and becomes the head.

``` c
static void insert_node(void *bp) {
    size_t index = find_index(GET_SIZE(HDRP(bp)));
    if (SEG_HEAD(index) == NULL) {
        SET_SEG_HEAD(index, bp); PUT_PTR(PREV(bp), bp); PUT_PTR(NEXT(bp), bp); return;
    }

    PUT_PTR(NEXT(bp), SEG_HEAD(index)); PUT_PTR(PREV(bp), PREV_FBLKP(SEG_HEAD(index)));
    PUT_PTR(NEXT(PREV_FBLKP(SEG_HEAD(index))), bp); PUT_PTR(PREV(SEG_HEAD(index)), bp);

    SET_SEG_HEAD(index, bp);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L140-L156"><code>mm.c</code> L140–L156</a> (reformatted to fit)</p>

Otherwise the new block is spliced in ahead of the current head: its `next` points at the old head and its `prev` at the old head's predecessor, which is the ring's tail; the tail's `next` and the old head's `prev` are then rewired to the new block, and the class head moves to it. Nothing is searched and nothing is allocated, so an insertion costs one `find_index` call and four pointer writes.

``` c
static void remove_node(void *bp) {
    size_t index = find_index(GET_SIZE(HDRP(bp)));

    char *cur_prev = PREV_FBLKP(bp);
    char *cur_next = NEXT_FBLKP(bp);

    PUT_PTR(NEXT(cur_prev), cur_next); PUT_PTR(PREV(cur_next), cur_prev);

    if (SEG_HEAD(index) == bp) {
        SET_SEG_HEAD(index, cur_next == bp ? NULL : cur_next);
    }
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L161-L174"><code>mm.c</code> L161–L174</a> (reformatted to fit)</p>

Removal is the same splice read the other way, and it needs no case analysis on position: `cur_prev` and `cur_next` come from the block's own two links, and the two neighbours are then pointed at each other. The only extra work is the class head, which moves to `cur_next` when the removed block was the head, and becomes `NULL` when the ring held that one node and `cur_next` is the block itself.

`find_fit` starts at the request's class and scans each non-empty ring in turn. If a class has no adequate block, it climbs to the next; once a class yields one or more fits, it keeps the **smallest** and stops. Each scan is linear in that ring's length, but allocated blocks are never visited:

``` c
static void *find_fit(size_t asize) {
    size_t minSize;
    size_t index = find_index(asize);
    char *ptr = NULL;

    while (index < CLASSNUM && ptr == NULL) {
        char *bp = SEG_HEAD(index);
        if (bp == NULL) {                        // (1)
            index++;
            continue;
        }
        do {
            if (asize <= GET_SIZE(HDRP(bp)) && ptr == NULL) {   // (2)
                ptr = bp;
                minSize = GET_SIZE(HDRP(ptr));
            } else if (asize <= GET_SIZE(HDRP(bp)) && ptr != NULL) {
                if (GET_SIZE(HDRP(bp)) < minSize) {
                    ptr = bp;
                    minSize = GET_SIZE(HDRP(ptr));
                }
            }
            bp = NEXT_FBLKP(bp);
        } while (bp != SEG_HEAD(index));         // (3)
        index++;
    }
    if (ptr != NULL)
        return ptr;

    /* No fit */
    return NULL;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L269-L301"><code>mm.c</code> L269–L301</a> (reformatted to fit)</p>

1.  Empty class? Climb to the next one immediately.
2.  Track the tightest block seen so far in this class.
3.  The list is circular, so we stop once we loop back to the head.

## Placement and splitting { data-toc-label="Placement" }

Once a fit is found, `place` marks it allocated and — if the leftover is at least one minimum block (16 bytes) — **splits** it, returning the tail to the free lists:

``` c
static void place(void *bp, size_t asize) {
    size_t csize = GET_SIZE(HDRP(bp));

    if ((csize - asize) >= (2*DSIZE)) {
        remove_node(bp);
        PUT(HDRP(bp), PACK(asize, GET_PREV_ALLOC(HDRP(bp)) | 0x1));
        PUT(HDRP(NEXT_BLKP(bp)), PACK(csize-asize, 0x2));     // (1)
        PUT(FTRP(NEXT_BLKP(bp)), PACK(csize-asize, 0x2)); insert_node(NEXT_BLKP(bp));
    } else {
        remove_node(bp);
        PUT(HDRP(bp), PACK(csize, GET_PREV_ALLOC(HDRP(bp)) | 0x1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp), PACK(GET_SIZE(HDRP(bp)), GET_ALLOC(HDRP(bp)) | 0x2)); // (2)
    }
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L180-L197"><code>mm.c</code> L180–L197</a> (reformatted to fit)</p>

1.  The remainder's header gets `PA = 1` (its new left neighbor — the block we just allocated — *is* allocated) and stays free, so it also gets a footer.
2.  No split: the following block's `PA` bit is set, because its left neighbor is now allocated. Both paths have to keep `PA` current, which is what makes the footer-less scheme work.

## mm_malloc: search, then grow { data-toc-label="mm_malloc" }

`mm_malloc` ties the pieces together: it rounds the request into a payload class, turns that payload into a block size, asks the free lists for a fit, and grows the heap only when nothing fits.

``` c
void *mm_malloc(size_t size) {
    size_t asize;           /* Adjusted block size */
    size_t extendsize;      /* Amount to extend heap if no fit */
    char *bp;

    /* Ignore spurious requests */
    if (size == 0)
        return NULL;

    /* Round the payload class, then add header overhead and alignment. */
    size = round_payload(size);
    if (size <= DSIZE)
        asize = 2*DSIZE;
    else
        asize = DSIZE * ((size + WSIZE + (DSIZE - 1)) / DSIZE);

    /* Search the free list for a fit */
    if ((bp = find_fit(asize)) != NULL) { place(bp, asize); return bp; }

    /* No fit found. Get more memory and place the block */
    extendsize = MAX(asize, CHUNKSIZE);
    if ((bp = extend_heap(extendsize/WSIZE)) == NULL)
        return NULL;

    place(bp, asize); return bp;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L336-L366"><code>mm.c</code> L336–L366</a> (reformatted to fit)</p>

`size <= DSIZE` is the floor case: the smallest block handed out is `2*DSIZE`, 16 bytes, because a free block must also hold two link words and a footer. Above that, `asize` is the payload plus a one-word header, rounded up to a multiple of `DSIZE`. When `find_fit` returns nothing, `MAX(asize, CHUNKSIZE)` decides how much to ask for, so a burst of small allocations does not turn into a burst of `mem_sbrk` calls. The new block comes back free and already linked into its class, so it goes straight to `place`.

Heap growth itself is one helper, shared with `mm_init`:

``` c
static void *extend_heap(size_t words) {
    char *bp; size_t size;

    /* Allocate an even number of words to maintain alignment */
    size = (words % 2)? (words + 1) * WSIZE : words * WSIZE;
    if ((bp = mem_sbrk(size)) == (void *)-1)
        return NULL;

    /* Initialize free block header/footer and the epilogue header */
    PUT(HDRP(bp), PACK(size, GET_PREV_ALLOC(HDRP(bp))));
    PUT(FTRP(bp), PACK(size, GET_PREV_ALLOC(HDRP(bp))));
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 0x1));

    /* Coalesce if the previous block was free */
    return coalesce(bp);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L246-L263"><code>mm.c</code> L246–L263</a> (reformatted to fit)</p>

`words` is rounded up to an even count so the new block stays 8-byte aligned. `mem_sbrk` returns the old break, which is exactly the payload address of the new block, and the epilogue header that used to sit at `HDRP(bp)` becomes the new block's header, inheriting its `PA` bit; the footer mirrors it. A fresh epilogue (`PACK(0, 0x1)`) is written after the block, with `PA` clear because the block below it is free until `place` allocates it. That is also why the helper ends in `coalesce`: an extension that lands next to a free tail merges into one large block instead of leaving two adjacent free blocks behind.

## Coalescing — immediate, four cases { data-toc-label="Coalescing" }

Freeing merges with adjacent free blocks *right away*. The `PA` bit and the next block's alloc bit give four cases:

<div class="grid cards" markdown>

-   **① prev alloc · next alloc**

    ---

    Nothing to merge — just insert the block into its class.

-   **② prev alloc · next free**

    ---

    Absorb the next block: `size += next`, rewrite header + footer.

-   **③ prev free · next alloc**

    ---

    Absorb backward into the previous block; the merged block starts at `PREV_BLKP(bp)`.

-   **④ prev free · next free**

    ---

    Swallow both neighbors into one block spanning all three.

</div>

Four cases, in this order: `prev alloc · next alloc` · `prev alloc · next free` · `prev free · next alloc` · `prev free · next free`.

``` c
static void *coalesce(void *bp)
{
    size_t prev_alloc = GET_PREV_ALLOC(HDRP(bp));            // (1)
    size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
    size_t size = GET_SIZE(HDRP(bp));

    if (prev_alloc && next_alloc) {                          // case 1
        insert_node(bp);
        return bp;
    }

    else if (prev_alloc && !next_alloc) {                    // case 2
        remove_node(NEXT_BLKP(bp));
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(bp), PACK(size, 0x2));
        PUT(FTRP(bp), PACK(size, 0x2));
    }

    else if (!prev_alloc && next_alloc) {                    // case 3
        remove_node(PREV_BLKP(bp));
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp), PACK(size, GET_PREV_ALLOC(HDRP(PREV_BLKP(bp)))));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, GET_PREV_ALLOC(HDRP(PREV_BLKP(bp)))));
        bp = PREV_BLKP(bp);
    }

    else {                                                   // case 4
        remove_node(PREV_BLKP(bp));
        remove_node(NEXT_BLKP(bp));
        size += GET_SIZE(HDRP(PREV_BLKP(bp))) + GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, GET_PREV_ALLOC(HDRP(PREV_BLKP(bp)))));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, GET_PREV_ALLOC(HDRP(PREV_BLKP(bp)))));
        bp = PREV_BLKP(bp);
    }

    insert_node(bp);                                         // (2)
    return bp;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L203-L240"><code>mm.c</code> L203–L240</a> (reformatted to fit)</p>

1.  Both neighbor-states come essentially for free: the previous one from *this* header's `PA` bit, the next one from the adjacent header — no linear scan.
2.  In every merging case the enlarged block is re-inserted once, into whichever class its new (larger) size selects.

Because free blocks are always merged on the spot, the heap never accumulates runs of tiny adjacent free blocks — the source of most fragmentation.

## Realloc: grow in place before copying { data-toc-label="Realloc" }

The naive `realloc` is "malloc a new block, `memcpy`, free the old." This implementation first tries to reuse the current block or its neighbors. Growing at the heap end or into the next block leaves the payload in place; using the previous block requires `memmove`, but still avoids allocating a separate replacement. A fresh malloc-copy-free cycle is the final fallback.

``` text
realloc(bp, size)
|
+- asize <= bsize ............... keep block in place
|  `- remainder >= 16 B ......... split off the tail
|
+- must grow -- try, in order:
   +- bp is the last block ....... mem_sbrk the shortage, extend in place
   +- next free hits heap end,
   |  but is still too small ...... absorb next + sbrk the rest
   +- next free, room to split ... absorb next, leave the remainder free
   +- next free, just enough ..... absorb the whole next block
   +- prev free & big enough ..... memmove into prev, absorb it
   +- both neighbors free enough . memmove into prev, absorb both
   `- none of the above .......... malloc + memcpy + free   (fallback)
```

Two touches make this pay off:

- **Slack on growth.** Before rounding, the request is padded by `size >> 4` (≈ 6 %). A program that grows a buffer in a loop then finds the next realloc already fits, avoiding a cascade of copies.
- **The heap-end shortcut.** If the block being grown is the last real block (its "next" is the epilogue, size 0), we simply `mem_sbrk` the exact shortfall and stretch the block, with no search and no copy.

Every move that keeps the payload in place is tried before the malloc-copy-free fallback:

``` c
void *mm_realloc(void *bp, size_t size)
{
    char *ptr;
    size_t asize;
    size_t bsize;
    size_t copySize;
    size_t prevSize;
    size_t nextSize;

    if (bp == NULL && size == 0)
        return bp;

    if (bp == NULL)
        return mm_malloc(size);

    if (size == 0) {
        mm_free(bp);
        return NULL;
    }

    bsize = GET_SIZE(HDRP(bp));
    /* Leave modest slack for future growth before payload-class rounding. */
    size += size >> 4;
    size = round_payload(size);
    if (size <= DSIZE)
        asize = 2*DSIZE;
    else
        asize = DSIZE * ((size + WSIZE + (DSIZE - 1)) / DSIZE);

    if (asize <= bsize) {
        if (bsize - asize >= (2*DSIZE)) {
            PUT(HDRP(bp), PACK(asize, GET_PREV_ALLOC(HDRP(bp)) | 0x1));
            PUT(HDRP(NEXT_BLKP(bp)), PACK(bsize-asize, 0x2));
            PUT(FTRP(NEXT_BLKP(bp)), PACK(bsize-asize, 0x2));
            PUT(HDRP(NEXT_BLKP(NEXT_BLKP(bp))),
                    PACK(GET_SIZE(HDRP(NEXT_BLKP(NEXT_BLKP(bp)))),
                            GET_ALLOC(HDRP(NEXT_BLKP(NEXT_BLKP(bp))))));
            if (!GET_ALLOC(HDRP(NEXT_BLKP(NEXT_BLKP(bp))))) {
                PUT(FTRP(NEXT_BLKP(NEXT_BLKP(bp))),
                        PACK(GET_SIZE(HDRP(NEXT_BLKP(NEXT_BLKP(bp)))),
                                GET_ALLOC(HDRP(NEXT_BLKP(NEXT_BLKP(bp))))));
            }
            coalesce(NEXT_BLKP(bp));
            return bp;
        }
        else {
            return bp;
        }
    }

    else {
        copySize = GET_SIZE(HDRP(bp)) - WSIZE;

        /* If this is the last real block, grow it directly with mem_sbrk. */
        if (GET_SIZE(HDRP(NEXT_BLKP(bp))) == 0) {           // (1)
            if (mem_sbrk(asize-bsize) == (void *)-1)
                return NULL;
            PUT(HDRP(bp), PACK(asize, GET_PREV_ALLOC(HDRP(bp)) | 0x1));
            PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 0x3));         // (2)
            return bp;
        }

        /* Prefer expanding into the next block because the payload stays put. */
        if (!GET_ALLOC(HDRP(NEXT_BLKP(bp)))) {
            ptr = NEXT_BLKP(bp);
            nextSize = GET_SIZE(HDRP(ptr));

            /* If the next free block reaches the heap end, absorb it and
             * extend only the remaining shortage.
             */
            if (GET_SIZE(HDRP(NEXT_BLKP(ptr))) == 0 && bsize + nextSize < asize) {
                if (mem_sbrk(asize - bsize - nextSize) == (void *)-1)
                    return NULL;
                remove_node(ptr);
                PUT(HDRP(bp), PACK(asize, GET_PREV_ALLOC(HDRP(bp)) | 0x1));
                PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 0x3));
                return bp;
            }

            if (nextSize >= (asize-bsize) + (4*DSIZE)) {
                remove_node(NEXT_BLKP(bp));
                PUT(HDRP(bp), PACK(asize, GET_PREV_ALLOC(HDRP(bp)) | 0x1));
                PUT(HDRP(NEXT_BLKP(bp)), PACK(nextSize-(asize-bsize), 0x2));
                PUT(FTRP(NEXT_BLKP(bp)), PACK(nextSize-(asize-bsize), 0x2));
                insert_node(NEXT_BLKP(bp));
                return bp;
            }
            else if (nextSize >= (asize-bsize)) {
                remove_node(NEXT_BLKP(bp));
                PUT(HDRP(bp), PACK(bsize+nextSize, GET_PREV_ALLOC(HDRP(bp)) | 0x1));
                PUT(HDRP(NEXT_BLKP(bp)), PACK(GET_SIZE(HDRP(NEXT_BLKP(bp))),
                        GET_ALLOC(HDRP(NEXT_BLKP(bp))) | 0x2));
                if (!GET_ALLOC(HDRP(NEXT_BLKP(bp)))) {
                    PUT(FTRP(NEXT_BLKP(bp)), PACK(GET_SIZE(HDRP(NEXT_BLKP(bp))),
                            GET_ALLOC(HDRP(NEXT_BLKP(bp))) | 0x2));
                }
                return bp;
            }
        }

        /* Expanding backward requires moving the payload into the previous block. */
        if (!GET_PREV_ALLOC(HDRP(bp))) {
            ptr = PREV_BLKP(bp);
            prevSize = GET_SIZE(HDRP(ptr));
            if (prevSize >= (asize-bsize) + (4*DSIZE)) {
                remove_node(ptr);
                memmove(ptr, bp, copySize);
                PUT(HDRP(ptr), PACK(asize, GET_PREV_ALLOC(HDRP(ptr)) | 0x1));
                bp = NEXT_BLKP(ptr);
                PUT(HDRP(bp), PACK(prevSize-(asize-bsize), 0x2));
                PUT(FTRP(bp), PACK(prevSize-(asize-bsize), 0x2));
                PUT(HDRP(NEXT_BLKP(bp)), PACK(GET_SIZE(HDRP(NEXT_BLKP(bp))),
                        GET_ALLOC(HDRP(NEXT_BLKP(bp)))));
                if (!GET_ALLOC(HDRP(NEXT_BLKP(bp)))) {
                    PUT(FTRP(NEXT_BLKP(bp)), PACK(GET_SIZE(HDRP(NEXT_BLKP(bp))),
                            GET_ALLOC(HDRP(NEXT_BLKP(bp)))));
                }
                coalesce(bp);
                return ptr;
            }
            else if (prevSize >= (asize-bsize)) {
                remove_node(ptr);
                memmove(ptr, bp, copySize);
                PUT(HDRP(ptr), PACK(bsize+prevSize, GET_PREV_ALLOC(HDRP(ptr)) | 0x1));
                bp = NEXT_BLKP(ptr);
                PUT(HDRP(bp), PACK(GET_SIZE(HDRP(bp)), GET_ALLOC(HDRP(bp)) | 0x2));
                if (!GET_ALLOC(HDRP(bp))) {
                    PUT(FTRP(bp), PACK(GET_SIZE(HDRP(bp)), GET_ALLOC(HDRP(bp)) | 0x2));
                }
                return ptr;
            }
        }

        /* If either neighbor alone is not enough, try using both sides. */
        if (!GET_PREV_ALLOC(HDRP(bp)) && !GET_ALLOC(HDRP(NEXT_BLKP(bp)))) {
            nextSize = GET_SIZE(HDRP(NEXT_BLKP(bp)));
            ptr = PREV_BLKP(bp);
            prevSize = GET_SIZE(HDRP(ptr));
            if (prevSize + nextSize >= (asize-bsize) + (4*DSIZE)) {
                remove_node(NEXT_BLKP(bp));
                remove_node(ptr);
                memmove(ptr, bp, copySize);
                PUT(HDRP(ptr), PACK(asize, GET_PREV_ALLOC(HDRP(ptr)) | 0x1));
                bp = NEXT_BLKP(ptr);
                PUT(HDRP(bp), PACK(prevSize+nextSize-(asize-bsize), 0x2));
                PUT(FTRP(bp), PACK(prevSize+nextSize-(asize-bsize), 0x2));
                insert_node(bp);
                return ptr;
            }
            else if (prevSize + nextSize >= (asize-bsize)) {
                remove_node(NEXT_BLKP(bp));
                remove_node(ptr);
                memmove(ptr, bp, copySize);
                PUT(HDRP(ptr), PACK(bsize+prevSize+nextSize,
                        GET_PREV_ALLOC(HDRP(ptr)) | 0x1));
                bp = NEXT_BLKP(ptr);
                PUT(HDRP(bp), PACK(GET_SIZE(HDRP(bp)), GET_ALLOC(HDRP(bp)) | 0x2));
                if (!GET_ALLOC(HDRP(bp))) {
                    PUT(FTRP(bp), PACK(GET_SIZE(HDRP(bp)), GET_ALLOC(HDRP(bp)) | 0x2));
                }
                return ptr;
            }
        }

        ptr = mm_malloc(size);
        if (ptr == NULL)
            return NULL;
        if (size < copySize)
            copySize = size;
        memcpy(ptr, bp, copySize);
        mm_free(bp);
        return ptr;
    }
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Malloc_Lab/malloclab-handout/mm.c#L395-L559"><code>mm.c</code> L395–L559</a> (reformatted to fit)</p>

1.  A zero-size next header is the epilogue: `bp` sits at the top of the heap.
2.  Re-lay the epilogue immediately after the grown block, with `PA = 1`.

When a neighbor *is* used, the code is deliberately stricter than the 16-byte structural minimum: it splits only when the remainder is at least `4·DSIZE = 32` bytes. A 16- or 24-byte remainder would be legal, but absorbing it trades a little internal slack for fewer small free-list nodes. Shrinking the current block still uses the ordinary `2*DSIZE` threshold; the 32-byte rule applies only to neighbor-assisted growth. When no adjacent path can satisfy the request, the code falls through to malloc-copy-free and copies at most the old payload's worth of bytes.
