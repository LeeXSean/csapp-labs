---
title: Bomb Lab
description: Defusing a binary bomb phase by phase — reading x86-64 assembly in GDB.
---

# Bomb Lab · Defusing the Bomb

<p class="article-meta">Reverse engineering <span class="dot">·</span> Keywords: GDB, x86-64, jump tables, recursion, BST <span class="dot">·</span> <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Bomb_Lab">Source · `bomb`</a></p>

!!! success "Verified locally"
    `./bomb psol.txt` defuses all **6 phases plus the secret stage** — ending on *"Congratulations! You've defused the bomb!"*

A binary that reads a line, checks it against something hidden, and calls `explode_bomb` if you're wrong. Six phases, plus a seventh you have to *discover*. The only tool that matters is GDB, and the only skill is reading intent out of assembly.

!!! abstract "The method"
    The bomb ships with no source for the phases — only the executable. Every phase follows the same anatomy, and so does the way in:

    1. `break phase_n`, `run`, then `disas` to see the phase's code.
    2. Find the conditional branch that guards the call to `explode_bomb`. The condition needed to **skip** that call is your constraint.
    3. Read the operands being compared — dump memory with `x/s`, `x/d`, print registers — and work **backward** to an input that satisfies them.

    A useful reflex: every `je/jne/jle/ja/...` that jumps *toward* `explode_bomb` is a wrong turn, so its negation points straight at the solution.

---

## Phase 1 · a plain string compare { data-toc-label="Phase 1" }

It hands your input and a fixed pointer to `strings_not_equal` (which, like `strcmp`, returns `0` when the two strings are equal) and explodes unless the result is zero:

``` asm
400ee0: sub    $0x8,%rsp
400ee4: mov    $0x402400,%esi          ; arg2 = a fixed string in .rodata
400ee9: call   401338 <strings_not_equal>   ; %eax = 0 iff input == that string
400eee: test   %eax,%eax               ; set flags from %eax
400ef0: je     400ef7 <phase_1+0x17>   ; %eax == 0 -> skip the bomb
400ef2: call   40143a <explode_bomb>
400ef7: add    $0x8,%rsp
400efb: ret
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Bomb_Lab"><code>bomb</code> · <code>phase_1</code> @ 400ee0–400efb</a></p>

So survival means `strings_not_equal` returns `0`, i.e. your line **equals the string at `0x402400`**. There's nothing to compute — just read that address in GDB:

``` gdb
(gdb) x/s 0x402400
0x402400: "Border relations with Canada have never been better."
```

> **`Border relations with Canada have never been better.`**

## Phase 2 · a loop over six numbers { data-toc-label="Phase 2" }

`read_six_numbers` parses six integers onto the stack. Two checks follow. First, the very first number is pinned to `1`:

``` asm
400f05: call   40145c <read_six_numbers>
400f0a: cmpl   $0x1,(%rsp)             ; numbers[0] must be 1
400f0e: je     400f30 <phase_2+0x34>   ; ok -> set up the loop
400f10: call   40143a <explode_bomb>
```

Then a loop walks the array with `%rbx` (a moving pointer) up to `%rbp` (one past the end), checking that each element is **twice** its predecessor — `add %eax,%eax` is just `%eax * 2`:

``` asm
400f17: mov    -0x4(%rbx),%eax         ; eax = previous element
400f1a: add    %eax,%eax               ; eax = previous * 2
400f1c: cmp    %eax,(%rbx)             ; current == previous * 2 ?
400f1e: je     400f25 <phase_2+0x29>
400f20: call   40143a <explode_bomb>
400f25: add    $0x4,%rbx               ; advance to the next int
400f29: cmp    %rbp,%rbx               ; reached the end?
400f2c: jne    400f17 <phase_2+0x1b>   ; no -> next element
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Bomb_Lab"><code>bomb</code> · <code>phase_2</code> @ 400f05–400f2c</a></p>

Starting from `1` and doubling five times gives a geometric sequence:

> **`1 2 4 8 16 32`**

## Phase 3 · a switch / jump table { data-toc-label="Phase 3" }

Here `sscanf` reads **two** integers (its format string lives at `0x4025cf` — inspect it and you'll see `"%d %d"`). The return value, the count of items parsed, must be at least `2`:

``` asm
400f47: lea    0xc(%rsp),%rcx          ; &second
400f4c: lea    0x8(%rsp),%rdx          ; &first
400f51: mov    $0x4025cf,%esi          ; "%d %d"
400f5b: call   400bf0 <__isoc99_sscanf@plt>
400f60: cmp    $0x1,%eax               ; the format string has two conversions
400f63: jg     400f6a <phase_3+0x27>   ; parsed > 1 value -> continue
400f65: call   40143a <explode_bomb>
```

The first number selects a branch through a **jump table** — the shape a `switch` takes once compiled. It is bounded to `0–7`, then used to index a table of code addresses at `0x402470`:

``` asm
400f6a: cmpl   $0x7,0x8(%rsp)          ; first must be <= 7 (unsigned compare)
400f6f: ja     400fad <phase_3+0x6a>   ; -> explode_bomb
400f71: mov    0x8(%rsp),%eax
400f75: jmp    *0x402470(,%rax,8)      ; goto table[first]
```

Each case loads a constant into `%eax`, and the final check demands the **second** number equal it. (Dump the table itself with `x/8a 0x402470` to read every case's target address.) Following the entry for `first = 1` lands on:

``` asm
400fb9: mov    $0x137,%eax             ; case 1 -> 0x137 (= 311)
400fbe: cmp    0xc(%rsp),%eax          ; second == 311 ?
400fc2: je     400fc9 <phase_3+0x86>   ; -> defused
400fc4: call   40143a <explode_bomb>
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Bomb_Lab"><code>bomb</code> · <code>phase_3</code> @ 400f47–400fc4</a></p>

Any of the eight cases yields a valid answer; taking case `1`:

> **`1 311`**

## Phase 4 · a recursive function { data-toc-label="Phase 4" }

`sscanf` again reads two integers; this time *exactly* two, and the first is capped at `14`:

``` asm
401029: cmp    $0x2,%eax               ; exactly two values parsed
40102c: jne    401035 <phase_4+0x29>   ; not exactly two -> explode_bomb
40102e: cmpl   $0xe,0x8(%rsp)          ; first <= 14
401033: jbe    40103a <phase_4+0x2e>   ; -> call func4
401035: call   40143a <explode_bomb>
```

The first number is passed to `func4(first, 0, 14)`, whose result must be `0`, and the second number must also be `0`:

``` asm
40103a: mov    $0xe,%edx               ; arg3 = hi = 14
40103f: mov    $0x0,%esi               ; arg2 = lo = 0
401044: mov    0x8(%rsp),%edi          ; arg1 = x = first
401048: call   400fce <func4>
40104d: test   %eax,%eax               ; func4 must return 0
40104f: jne    401058 <phase_4+0x4c>   ; -> explode_bomb
401051: cmpl   $0x0,0xc(%rsp)          ; second == 0
401056: je     40105d <phase_4+0x51>   ; -> defused
```

`func4` is a **binary search** over `[lo, hi]`. It computes the midpoint (the `shr`/`sar` pair is just a signed divide-by-two that rounds toward zero), then recurses into one half — doubling the running result on the way back:

``` asm
400fce: sub    $0x8,%rsp
400fd2: mov    %edx,%eax
400fd4: sub    %esi,%eax               ; hi - lo
400fd6: mov    %eax,%ecx               ; the shr/sar pair rounds the
400fd8: shr    $0x1f,%ecx              ;   difference toward zero
400fdb: add    %ecx,%eax
400fdd: sar    $1,%eax
400fdf: lea    (%rax,%rsi,1),%ecx      ; mid = lo + (hi-lo)/2
400fe2: cmp    %edi,%ecx
400fe4: jle    400ff2 <func4+0x24>     ; mid <= x -> check for the match
400fe6: lea    -0x1(%rcx),%edx         ; x < mid -> recurse on [lo, mid-1]
400fe9: call   400fce <func4>
400fee: add    %eax,%eax               ; result = 2*r
400ff0: jmp    401007 <func4+0x39>
400ff2: mov    $0x0,%eax
400ff7: cmp    %edi,%ecx               ; x == mid -> return 0
400ff9: jge    401007 <func4+0x39>
400ffb: lea    0x1(%rcx),%esi          ; x > mid -> recurse on [mid+1, hi]
400ffe: call   400fce <func4>
401003: lea    0x1(%rax,%rax,1),%eax   ; result = 2*r+1
401007: add    $0x8,%rsp
40100b: ret
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Bomb_Lab"><code>bomb</code> · <code>phase_4</code> @ 401029–401056 and <code>func4</code> @ 400fce–40100b</a></p>

The result stays `0` only along a path that never takes the `2*r + 1` (go-right) branch. So `func4` returns `0` exactly for the midpoints reached by going left from `[0, 14]`:

``` text
   func4(x, 0, 14),  mid = (lo + hi) / 2 at each step:
       x == mid  ->  return 0
       x <  mid  ->  go left,  result = 2*r        (0 stays 0)
       x >  mid  ->  go right, result = 2*r + 1     (never 0 again)

   Follow the all-left spine; the matching x at each level returns 0:

       [0,14]  mid = 7   --  x = 7  -->  0
         |
         +--  [0,6]  mid = 3   --  x = 3  -->  0
                |
                +--  [0,2]  mid = 1   --  x = 1  -->  0
                       |
                       +--  [0,0]  mid = 0   --  x = 0  -->  0

   => func4 returns 0 for x in {0, 1, 3, 7}
```

Take `1`, and pair it with the required second `0`:

> **`1 0`** — and note what appending `DrEvil` here unlocks in the [secret phase](#the-secret-phase).

## Phase 5 · a table-lookup cipher { data-toc-label="Phase 5" }

The input must be **6 characters** long. Then each character is transformed and the six results must spell a target word. The transform: mask a character down to its **low 4 bits**, and use that as an index into a 16-byte table at `0x4024b0` — the string `"maduiersnfotvbyl"` (peek with `x/s 0x4024b0`):

``` asm
40108b: movzbl (%rbx,%rax,1),%ecx      ; ecx = input[i]
40108f: mov    %cl,(%rsp)
401092: mov    (%rsp),%rdx
401096: and    $0xf,%edx               ; keep the low nibble
401099: movzbl 0x4024b0(%rdx),%edx     ; edx = table[nibble]
4010a0: mov    %dl,0x10(%rsp,%rax,1)   ; append to the result buffer
4010a4: add    $0x1,%rax               ; next character
4010a8: cmp    $0x6,%rax
4010ac: jne    40108b <phase_5+0x29>   ; loop over all six
4010ae: movb   $0x0,0x16(%rsp)         ; NUL-terminate the result
4010b3: mov    $0x40245e,%esi          ; target = "flyers"
4010b8: lea    0x10(%rsp),%rdi
4010bd: call   401338 <strings_not_equal>
4010c2: test   %eax,%eax
4010c4: je     4010d9 <phase_5+0x77>   ; equal -> defused
4010c6: call   40143a <explode_bomb>
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Bomb_Lab"><code>bomb</code> · <code>phase_5</code> @ 40108b–4010c6</a></p>

So the puzzle is: pick six bytes whose low nibbles index the letters of `"flyers"`. Reading the table, those letters sit at indices `9, 15, 14, 5, 6, 7`. Any characters with those low nibbles work — a printable set is one per column:

| target letter | `f` | `l` | `y` | `e` | `r` | `s` |
|---------------|-----|-----|-----|-----|-----|-----|
| table index (= low nibble needed) | 9 | 15 | 14 | 5 | 6 | 7 |
| a printable char with that low nibble | `)` `0x29` | `/` `0x2f` | `.` `0x2e` | `%` `0x25` | `&` `0x26` | `'` `0x27` |

> **`)/.%&'`**

## Phase 6 · reordering a linked list { data-toc-label="Phase 6" }

It reads six numbers and enforces two properties: each is in `1–6`, and all are **distinct** (a nested loop compares every pair):

``` asm
401119: mov    0x0(%r13),%eax
40111d: sub    $0x1,%eax
401120: cmp    $0x5,%eax               ; (value - 1) <= 5  -> value in 1..6
401123: jbe    401128 <phase_6+0x34>
401125: call   40143a <explode_bomb>
...                                    ; loop bookkeeping for the nested scan:
                                       ;   401128-401132 advance the outer index,
                                       ;   401135-40114d walk the inner one
40113b: cmp    %eax,0x0(%rbp)          ; compare against every other value
40113e: jne    401145 <phase_6+0x51>   ; must differ -> distinct
401140: call   40143a <explode_bomb>
```

Next it maps every value `x` to `7 - x`, and uses those to index into a six-node **linked list** in `.data` (each node is `{ int value; int pad; node *next; }`). It threads the nodes into the order your numbers specify, then verifies that order is **descending by node value**:

``` asm
40115b: mov    $0x7,%ecx
401160: mov    %ecx,%edx
401162: sub    (%rax),%edx             ; 7 - each input, written back in place
401164: mov    %edx,(%rax)
...                                    ; loop bookkeeping, 401166-401174: step %rax
                                       ;   through the six words, then start the walk
401176: mov    0x8(%rdx),%rdx          ; follow ->next to the chosen node
40117a: add    $0x1,%eax
40117d: cmp    %ecx,%eax
40117f: jne    401176 <phase_6+0x82>   ; until the 7-x'th node
...                                    ; pointer store into 0x20(%rsp,...) and the
                                       ;   list-relinking pass, 401181-4011d2
4011df: mov    0x8(%rbx),%rax
4011e3: mov    (%rax),%eax
4011e5: cmp    %eax,(%rbx)
4011e7: jge    4011ee <phase_6+0xfa>   ; node.value >= next.value -> descending
4011e9: call   40143a <explode_bomb>
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Bomb_Lab"><code>bomb</code> · <code>phase_6</code> @ 401119–4011e9</a></p>

So the recipe is: read the node values, sort them **descending**, and emit `7 - node` for each. Dumping the list (`x/12xw 0x6032d0` shows each node's value and `next`):

``` text
   node    1      2      3      4      5      6
   value   0x14c  0xa8   0x39c  0x2b3  0x1dd  0x1bb

   sort the values high -> low; that fixes the node order,
   and the input is 7 - node:

   value   0x39c  0x2b3  0x1dd  0x1bb  0x14c  0xa8
   node    3      4      5      6      1      2
   input   4      3      2      1      6      5
```

> **`4 3 2 1 6 5`**

---

## The secret phase

There's a seventh phase, and finding it is half the puzzle. After all six are defused, `phase_defused` quietly re-parses the line you gave **phase 4** — this time as `"%d %d %s"` — and checks whether the trailing word is `"DrEvil"`:

``` asm
4015d8: cmpl   $0x6,0x202181(%rip)     ; num_input_strings == 6: all phases done
4015df: jne    40163f <phase_defused+0x7b>
4015f0: mov    $0x402619,%esi          ; "%d %d %s"
4015f5: mov    $0x603870,%edi          ; phase 4's saved input line
4015fa: call   400bf0 <__isoc99_sscanf@plt>
4015ff: cmp    $0x3,%eax               ; the %s must have been filled
401602: jne    401635 <phase_defused+0x71>
401604: mov    $0x402622,%esi          ; "DrEvil"
401609: lea    0x10(%rsp),%rdi         ; the %s captured from phase 4's line
40160e: call   401338 <strings_not_equal>
401613: test   %eax,%eax
401615: jne    401635 <phase_defused+0x71>
401630: call   401242 <secret_phase>   ; match -> secret phase
```

So phase 4's answer grows a third token: **`1 0 DrEvil`**. The secret phase then reads one integer (`1 <= n <= 1001`, from a `(n-1) <= 0x3e8` check) and calls `fun7` on a binary search tree rooted at `0x6030f0`; the return value must be exactly `2`:

``` asm
401204: sub    $0x8,%rsp
401208: test   %rdi,%rdi               ; NULL child -> return -1
40120b: je     401238 <fun7+0x34>
40120d: mov    (%rdi),%edx             ; node value
40120f: cmp    %esi,%edx               ; compare node value with input n
401211: jle    401220 <fun7+0x1c>      ; node <= n -> match or go right
401213: mov    0x8(%rdi),%rdi          ; n < node -> left child
401217: call   401204 <fun7>
40121c: add    %eax,%eax               ; result = 2*r
40121e: jmp    40123d <fun7+0x39>
401220: mov    $0x0,%eax
401225: cmp    %esi,%edx
401227: je     40123d <fun7+0x39>      ; n == node -> return 0
401229: mov    0x10(%rdi),%rdi         ; n > node -> right child
40122d: call   401204 <fun7>
401232: lea    0x1(%rax,%rax,1),%eax   ; result = 2*r+1
401236: jmp    40123d <fun7+0x39>      ; -> return with that value
401238: mov    $0xffffffff,%eax        ; sentinel for a NULL child
40123d: add    $0x8,%rsp
401241: ret
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Bomb_Lab"><code>bomb</code> · <code>phase_defused</code> @ 4015d8–401630 and <code>fun7</code> @ 401204–401241</a></p>

`fun7` **encodes the path it walks** into its return value: each step left multiplies by `2`, each step right does `2·r + 1`, and a match returns `0`. Read backward, a result of `2` factors uniquely as `2 = 2·(2·0 + 1)` — that is, *left, then right, then a match*:

``` text
   fun7 walks a BST; each step folds into the return value:
       n <  node  ->  go left,   return 2*r
       n >  node  ->  go right,  return 2*r + 1
       n == node  ->  match,     return 0

   Want return = 2.  Factor it to read the path back off:

       2  =  2 * ( 2*0 + 1 )
       |        |      `-- match          -> 0
       |        `--------- step RIGHT     -> 2*0 + 1
       `------------------ step LEFT      -> 2*1

   So the path is  left, then right, then match:

          root (0x6030f0)
          /
       node               (n < root  -> left)
          \
         [ 22 ]           (n > node  -> right, and n == 22 -> match)

   =>  n = the value at  root -> left -> right  =  22
```

So `n` must equal the value stored at *root -> left -> right*, which is `22` — the one number that threads exactly the `left, right, match` path:

> **`22`**
