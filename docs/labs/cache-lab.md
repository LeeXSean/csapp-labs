---
title: Cache Lab
description: An LRU cache simulator, and a blocked matrix transpose tuned to a tiny direct-mapped cache.
---

# Cache Lab · Dancing With the Cache

<p class="article-meta">Memory hierarchy <span class="dot">·</span> Keywords: LRU, conflict misses, blocking <span class="dot">·</span> <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Cache_Lab/cachelab-handout">csim.c · trans.c</a></p>

!!! success "Verified locally"
    `./driver.py` → **53 / 61**: csim **27/27**, transpose misses **288 / 1180 / 1993** for 32×32 / 64×64 / 61×67.

Part A asks you to model a cache; Part B asks you to get a real transpose under a miss budget. Between them the miss count stops being an abstraction.

!!! abstract "The assignment"
    - **Part A — `csim.c`.** Write a cache simulator that takes `(-s, -E, -b)` — set-index bits, lines per set, block-offset bits — replays a `valgrind` memory trace, and reports **hits, misses, evictions**, with LRU replacement.
    - **Part B — `trans.c`.** Transpose a matrix `B = A^T`, scored purely by the misses it causes on a **1 KB direct-mapped cache with 32-byte blocks** (32 sets, one line each, 8 `int`s per block). Lower is better.

---

## Part A · Simulating the cache { data-toc-label="Part A" }

### Anatomy of an address

A cache never sees "a variable" — only an address, split into three fields. The block offset is irrelevant to a hit/miss decision, so the simulator computes just the **set index** and the **tag**:

``` text
   63                       b+s        b            0
  +---------------------------+---------+-----------+
  |            tag            |  set    |  offset   |
  +---------------------------+---------+-----------+
   address >> (b+s)            s bits     b bits
```

`manipulate` shifts the address right by `b`, masks to `s` bits for `set_index`, and keeps everything above the set field, `address >> (b + s)`, as the `tag` that identifies which block currently lives in a line.

### The cache in memory

Each line stores just enough to answer "is this block here, and how recently was it used?" The cache is an array of sets, each a lazily-allocated array of `E` lines:

``` c
typedef struct {
    uint8_t valid; uint64_t tag; uint32_t LRU_counter;
} cache_line;
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/csim.c#L9-L14"><code>csim.c</code> L9–L14</a> (reformatted to fit)</p>

### Hit, miss, evict

`manipulate` decodes the address and looks through the set. A matching valid line is a **hit**; otherwise it's a **miss** that must load the block — filling an empty line if one exists, or **evicting** the least-recently-used line if the set is full:

``` c
void manipulate(char identifier, uint64_t address, uint32_t size) {
    uint32_t set_index = (address >> b) & ((0x1 << s) - 1);
    uint64_t tag = address >> (b + s);

    if (verbose && M_unique) {
        printf("%c %lx,%d ", identifier, address, size);
        M_unique = 0;
    }

    // If the set is empty, cache miss — simulate loading the block into cache
    if (cache[set_index] == NULL) {
        cache[set_index] = (cache_line*)malloc(sizeof(cache_line) * E);
        // Initialize to prevent segfault
        for (int i = 0; i < E; i++) {
            cache[set_index][i].valid = 0;
            cache[set_index][i].tag = 0;
            cache[set_index][i].LRU_counter = 0;
        }
        insert(set_index, tag);
    }
    // If the set is not empty, check valid bit and tag
    else {
        for (int i = 0; i < E; i++) {
            if (cache[set_index][i].valid && cache[set_index][i].tag == tag) {
                hits++;
                cache[set_index][i].LRU_counter = 1;
                update(set_index);
                if (verbose) {
                    printf("hit ");
                }
                return;
            }
        }
        // Valid bit is 0, or tag mismatch — cache miss
        insert(set_index, tag);
    }
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/csim.c#L31-L75"><code>csim.c</code> L31–L75</a> (reformatted to fit)</p>

The hit path sets the line's `LRU_counter` back to 1, then `update` bumps every valid line of the set, this one included, so the line just used ends up with the smallest counter. A miss falls out of the loop into `insert`, which counts it and takes the first line whose `valid` bit is 0:

``` c
void insert(uint32_t set_index, uint64_t tag) {
    misses++;
    if (verbose) {
        printf("miss ");
    }

    cache_line* set = cache[set_index];
    for (int i = 0; i < E; i++) {
        if (!set[i].valid) {
            set[i].valid = 1;
            set[i].tag = tag;
            set[i].LRU_counter = 1;
            update(set_index); return;
        }
    }
    // Cache set is full, evict the LRU line and load the new one
    evict(set_index, tag);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/csim.c#L78-L100"><code>csim.c</code> L78–L100</a> (reformatted to fit)</p>

If every line is already valid, `insert` hands the set to `evict`, which scans for the **largest** `LRU_counter` — the line untouched the longest — and overwrites it with the new tag:

``` c
void evict(uint32_t set_index, uint64_t tag) {
    uint32_t replace_index = 0;
    uint32_t max_LRU;

    evictions++;
    if (verbose) {
        printf("eviction ");
    }

    cache_line* set = cache[set_index];
    max_LRU = set[0].LRU_counter;
    for (int i = 0; i < E; i++) {
        if (set[i].LRU_counter > max_LRU) {
            max_LRU = set[i].LRU_counter;
            replace_index = i;
        }
    }
    set[replace_index].valid = 1;
    set[replace_index].tag = tag;
    set[replace_index].LRU_counter = 1;
    update(set_index);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/csim.c#L103-L128"><code>csim.c</code> L103–L128</a> (reformatted to fit)</p>

### LRU without a global scan

The replacement clock. Every access **ages the current set** by bumping the counter of each valid line, after resetting the just-used line to the minimum. So the most-recently-used line always carries the smallest counter and the LRU victim the largest:

``` c
void update(uint32_t set_index) {
    cache_line* set = cache[set_index];
    for (int j = 0; j < E; j++) {
        if (set[j].valid) {
            set[j].LRU_counter++;
        }
    }
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/csim.c#L136-L146"><code>csim.c</code> L136–L146</a> (reformatted to fit)</p>

Because all activity for one access is confined to a single set, only that set is walked — `E` lines per access instead of `S·E` — and no global timestamp update is ever needed.

### Driving it

`load_trace` parses each `" %c %lx,%d"` line. The one wrinkle is the `M` (modify) operation — a load immediately followed by a store — so it drives the cache **twice**; `L` and `S` once; `I` (instruction fetch) is ignored:

``` c
void load_trace() {
    FILE* pFile = fopen(trace_file, "r");
    if (!pFile) return;

    char identifier; uint64_t address; uint32_t size;

    while(fscanf(pFile, " %c %lx,%d", &identifier, &address, &size) > 0) {
        M_unique = 1;
        switch (identifier) {
        case 'I':
            break;
        case 'M':
            // M operation is a read followed by a write — process twice
            manipulate(identifier, address, size);
            manipulate(identifier, address, size); break;
        case 'L':
        case 'S':
            manipulate(identifier, address, size); break;
        }
        if (verbose && identifier != 'I') printf("\n");
    }
    fclose(pFile);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/csim.c#L149-L178"><code>csim.c</code> L149–L178</a> (reformatted to fit)</p>

`main` is only the driver: `getopt` reads `-s -E -b -t`, `S = 0x1 << s` sizes the set array, `load_trace()` replays the file, then each set and the outer array are freed and `printSummary` reports the counters — [csim.c L181–L234](https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/csim.c#L181-L234).

---

## Part B · Transpose under a hostile cache { data-toc-label="Part B" }

### The cache that fights back { data-toc-label="Cache geometry" }

Transposing is trivial arithmetic; the whole difficulty is spatial. The grading cache has **32 sets**, one line each, **8 `int`s per block**. A `32×32` matrix row is 32 `int`s = 4 blocks, so **8 rows** fill the entire cache — meaning rows `i` and `i+8` collide in the same sets. For `64×64`, a row is 8 blocks, so only **4 rows** fill the cache and rows `i` and `i+4` collide:

| Matrix | `int`s / row | blocks / row | rows to fill cache | rows that **conflict** |
|--------|-------------|--------------|--------------------|-----------------------|
| 32×32 | 32 | 4 | 8 | `i` and `i+8` |
| 64×64 | 64 | 8 | 4 | `i` and `i+4` |

Worse, `A` and `B` are laid out so that `A[k][k]` and `B[k][k]` land in the **same set** — the diagonal elements fight each other. Everything below is about dodging these two collisions.

### 32×32 — block by 8, read before you write { data-toc-label="32×32" }

Process the matrix in `8×8` tiles (one tile row = one cache block). The move that matters: read all **8** values of an `A` row into local variables *first*, then write them down a column of `B`. The eight registers break the read/write interleaving, so `A`'s cache block is fully consumed before any `B` write can evict it:

Three branches: the 32×32 tiles · the 64×64 four-quadrant passes · the general `16×16` blocked fallback.

``` c
void transpose_submit(int M, int N, int A[N][M], int B[M][N]) {
    int i, j, k, v1, v2, v3, v4, v5, v6, v7, v8;
    if (M == 32 && N == 32) {
        for (i = 0; i < N; i += 8) {
            for (j = 0; j < M; j += 8) {
                for (k = i; k < i + 8; k++) {
                    v1 = A[k][j];     v2 = A[k][j+1];
                    v3 = A[k][j+2];   v4 = A[k][j+3];
                    v5 = A[k][j+4];   v6 = A[k][j+5];
                    v7 = A[k][j+6];   v8 = A[k][j+7];

                    B[j][k] = v1;     B[j+1][k] = v2;
                    B[j+2][k] = v3;   B[j+3][k] = v4;
                    B[j+4][k] = v5;   B[j+5][k] = v6;
                    B[j+6][k] = v7;   B[j+7][k] = v8;
                }
            }
        }
    } else if (M == 64 && N == 64) {
        for (i = 0; i < N; i += 8) {
            for (j = 0; j < M; j += 8) {
                for (k = i; k < i + 4; k++) {
                    v1 = A[k][j]; v2 = A[k][j+1]; v3 = A[k][j+2]; v4 = A[k][j+3];
                    v5 = A[k][j+4]; v6 = A[k][j+5]; v7 = A[k][j+6]; v8 = A[k][j+7];

                    B[j][k] = v1;       B[j][k+4] = v5;
                    B[j+1][k] = v2;     B[j+1][k+4] = v6;
                    B[j+2][k] = v3;     B[j+2][k+4] = v7;
                    B[j+3][k] = v4;     B[j+3][k+4] = v8;

                }

                for (k = j; k < j + 4; k++) {
                    v1 = A[i+4][k]; v2 = A[i+5][k]; v3 = A[i+6][k]; v4 = A[i+7][k];
                    v5 = B[k][i+4]; v6 = B[k][i+5]; v7 = B[k][i+6]; v8 = B[k][i+7];

                    B[k][i+4] = v1; B[k][i+5] = v2; B[k][i+6] = v3; B[k][i+7] = v4;
                    B[k+4][i] = v5;    B[k+4][i+1] = v6;
                    B[k+4][i+2] = v7;  B[k+4][i+3] = v8;
                }

                for (k = i + 4; k < i + 8; k++) {
                    v1 = A[k][j+4]; v2 = A[k][j+5]; v3 = A[k][j+6]; v4 = A[k][j+7];

                    B[j+4][k] = v1;
                    B[j+5][k] = v2;
                    B[j+6][k] = v3;
                    B[j+7][k] = v4;
                }
            }
        }
    } else {
        for (i = 0; i < N; i += 16) {
            for (j = 0; j < M; j += 16) {
                for (v1 = i; v1 < i + 16 && v1 < N; v1++) {
                    for (v2 = j; v2 < j + 16 && v2 < M; v2++) {
                        B[v2][v1] = A[v1][v2];
                    }
                }
            }
        }
    }
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/trans.c#L23-L102"><code>trans.c</code> L23–L102</a> (reformatted to fit)</p>

Each `k` iteration therefore reads one full `A` block — the first read misses, the next seven hit — and only then writes the eight values down column `k` of `B`. On a diagonal tile the write target `B[k][k]` lands in the same set as the `A` block being read, and consuming that block first is what keeps the conflict to a single unavoidable miss instead of one per element.

### 32×32, take two — defer the diagonal { data-toc-label="32×32 · one temp" }

`trans.c` holds a second, never-registered version of the same idea — `transpose_32`. It goes back to plain element-wise copying and keeps a **single** temporary; the only trick is that on diagonal tiles the diagonal element is held back until its row is finished:

``` c
void transpose_32(int M, int N, int A[N][M], int B[M][N]) {
    int i, j, i1, j1, tmp;
    for (i = 0; i < N; i += 8) {
        for (j = 0; j < M; j += 8) {
            for (i1 = i; i1 < i + 8; i1++) {
                for (j1 = j; j1 < j + 8; j1++) {
                    if (j == i && j1 == i1) {
                        tmp = A[j1][j1];
                    } else {
                        B[j1][i1] = A[i1][j1];
                    }
                }
                if (i == j) {
                    B[i1][i1] = tmp;
                }
            }
        }
    }
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/trans.c#L110-L137"><code>trans.c</code> L110–L137</a> (reformatted to fit)</p>

Inside a diagonal tile the target `B[i1][i1]` lives in the same set as the `A` row being read, so writing it mid-row would evict that row; the loop parks `A[j1][j1]` in `tmp` and stores it once the row is finished, which pays the conflict exactly once.

"Read all eight into registers" and "don't touch the conflicting line until the row is done" are two answers to the same eviction, and the cache can't tell them apart: run through `csim-ref`, this version scores **288 misses — hits, misses, and evictions all identical to the submission**. Same score, an eighth of the temporaries; it just was never wired into `registerFunctions`, so the driver never saw it. A third version, `transpose_64`, sits below it in the same file and shares that fate — the 64×64 code the driver actually scores is the branch inline in `transpose_submit`.

### 64×64 — the four-quadrant shuffle { data-toc-label="64×64" }

Here `8×8` blocking self-destructs: within a tile, rows `k` and `k+4` conflict. Treat each `8×8` tile as **four `4×4` quadrants** and use `B`'s own upper-right quadrant as a scratch buffer, so data is staged in cache-friendly `4×4` pieces. The numbered labels below always refer to quadrants of **`A`**; transposition changes where each one belongs in `B`:

``` text
   A source                         B after pass 1
   +--------+--------+              +--------+--------+
   |  (1)   |  (2)   |              | (1)^T  | (2)^T* |
   +--------+--------+              +--------+--------+
   |  (3)   |  (4)   |              |   ?    |   ?    |
   +--------+--------+              +--------+--------+

   B after pass 2                   B final (after pass 3)
   +--------+--------+              +--------+--------+
   | (1)^T  | (3)^T  |              | (1)^T  | (3)^T  |
   +--------+--------+              +--------+--------+
   | (2)^T  |   ?    |              | (2)^T  | (4)^T  |
   +--------+--------+              +--------+--------+

   * (2)^T is parked in B's upper-right; its final home is lower-left.
```

The transpose runs in three passes.

1.  **Top half of `A`, with (2) parked.** Read `A`'s rows `i..i+3` (quadrants (1)(2)). Write `(1)^T` into `B`'s upper-left, and park `(2)^T` in `B`'s upper-right — a temporary home, since its final position is lower-left.
2.  **Put both off-diagonal quadrants in their final homes.** For each column, read `A`'s quadrant (3) *and* the parked `(2)^T` values. Overwrite `B`'s upper-right with `(3)^T`, then move the parked `(2)^T` down into `B`'s lower-left. One column at a time keeps every touched line resident.
3.  **Finish the bottom-right.** A straight `4×4` transpose writes `(4)^T` into the only empty quadrant.

The payoff: what would be a storm of conflict misses becomes a handful, because no pass ever keeps two conflicting rows live at once.

The code for these three passes is the `else if (M == 64 && N == 64)` branch of `transpose_submit`, printed once in the [32×32 section](#3232-block-by-8-read-before-you-write), where the full function is listed; in [`trans.c` L47–L85](https://github.com/LeeXSean/csapp-labs/blob/main/Cache_Lab/cachelab-handout/trans.c#L47-L85) it is the same range. Reading it against the loops above: `for (k = i; k < i + 4; k++)` is pass 1, `for (k = j; k < j + 4; k++)` is pass 2, and `for (k = i + 4; k < i + 8; k++)` is pass 3. The file carries a standalone `transpose_64` with the identical statements and the quadrant roles written out as comments; it is not registered, so the branch above is what the driver scores.
### The general case

For any other `M×N`, correctness matters more than the last few misses, so `transpose_submit` falls through to its `else` branch: a plain `16×16` blocked transpose with bounds guards. `i` and `j` step by 16, `v1` and `v2` are reused as the inner indices, and the `v1 < N` / `v2 < M` tests cut the last partial tile.
