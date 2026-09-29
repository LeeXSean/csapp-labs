---
title: Data Lab
description: Integer and floating-point operations from bitwise primitives only — a walkthrough of Data Lab.
---

# Data Lab · Fighting With Bits

<p class="article-meta">Data representation <span class="dot">·</span> Keywords: bitwise ops, two's complement, IEEE 754 <span class="dot">·</span> <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c">bits.c</a></p>

!!! success "Verified locally"
    `./btest` → **36 / 36** correct · `./dlc bits.c` reports the solution legal (`-m32`).

Data Lab is the course's first assignment, and a deliberately constrained one: under a strict set of rules, you re-implement everyday operations using nothing but raw bit manipulation.

!!! note "The rules"
    **Integer problems** may use only `! ~ & ^ | + << >>`, each with an operator budget; no `if` / loops / `==` / `*` / casts / constants larger than `0xFF`. **Floating-point problems** relax to allow loops and conditionals, but still forbid any float type or operation — you treat a `float` as a 32-bit `unsigned` and manhandle its bits directly.

The challenge is never "get it right" — it's "get it right *within budget*." Below are the most instructive problems in detail; the rest follow the same playbook.

---

## Integer Problems

### isTmax — recognizing the maximum without comparing { data-toc-label="isTmax" }

Decide whether `x` is the two's-complement max, `0x7FFFFFFF`. With no `==`, translate "equal" into "XOR to zero."

``` c
int isTmax(int x) {
  return !!(x ^ ~0) & !((x + 1) ^ ~x);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c#L165-L167"><code>bits.c</code> L165–L167</a> (reformatted to fit)</p>

Key observation: `Tmax + 1` overflows to `Tmin (0x80000000)`, and `~Tmax` is *also* `0x80000000`. So **`x` is Tmax iff `x + 1 == ~x`**, expressed as `(x+1) ^ ~x == 0`. The trap is that `x = -1 (0xFFFFFFFF)` satisfies it too (both sides are 0), which is what the leading `!!(x ^ ~0)` removes: `x ^ ~0` is just `~x`, and `~(-1)` is 0. The same recipe — turn equality into XOR-to-zero, then exclude the false positive — recurs throughout Data Lab.

### isAsciiDigit — range checks via the sign bit { data-toc-label="isAsciiDigit" }

Test whether `0x30 ≤ x ≤ 0x39`. With no `<=`, split the range into "are the high bits right?" plus "did the low nibble overflow?"

``` c
int isAsciiDigit(int x) {
  return !(x >> 4 ^ 3) & !(9 + ~(x ^ 48) + 1 >> 31);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c#L201-L203"><code>bits.c</code> L201–L203</a> (reformatted to fit)</p>

Every digit `'0'..'9'` has its high nibble equal to `0x3`, so `x >> 4 ^ 3` is zero exactly inside `0x30..0x3F` and the first `!` tests that. The second factor turns the sign bit into a comparison: `~(x ^ 48) + 1` is `-(x ^ 0x30)`, and with the high nibble already known to be `0x3`, `x ^ 0x30` extracts exactly the low digit `d`, so `9 + ~(x ^ 48) + 1` evaluates to `9 - d`. That is non-negative while `d ≤ 9` and negative from `d ≥ 10`; `>> 31` takes the sign bit and the outer `!` turns it into "is it ≤ 9?"

!!! tip "The recurring trick: arithmetic `>> 31` = extract the sign"
    On a 32-bit two's-complement value, `x >> 31` smears the sign bit across the whole word: `0x00000000` for non-negative, `0xFFFFFFFF` for negative. It doubles as both a **sign test** and an **all-zeros/all-ones mask generator** — the master key of this lab.

### conditional — forging a mask from a boolean { data-toc-label="conditional" }

Implement `x ? y : z` without `?:`. The idea: turn "is `x` truthy?" into an all-ones-or-all-zeros mask, then use it to pick `y` or `z`.

``` c
int conditional(int x, int y, int z) {
  return ((~!x + 1 ^ y) & y) ^ (~!x + 1 & z);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c#L211-L213"><code>bits.c</code> L211–L213</a> (reformatted to fit)</p>

`!x` collapses any nonzero to `0` and `0` to `1`; negating that with `~!x + 1` gives `mask = 0x00000000` for a truthy `x` and `mask = 0xFFFFFFFF` for a falsy one, and the expression writes the mask out twice, once per operand. Substituting the two extremes verifies it: with `mask = 0` the result is `(0 ^ y) & y ^ (0 & z) = y`, and with `mask = ~0` it is `(~y & y) ^ z = 0 ^ z = z`.

"Boolean → all-ones/all-zeros mask → pick one of two" is the universal recipe for branch-like problems — `isLessOrEqual` below rests on the same idea.

### isLessOrEqual — comparison that dodges overflow { data-toc-label="isLessOrEqual" }

The naive `x <= y` checks `y - x >= 0`, but subtracting operands of opposite sign can overflow. The fix is to **split into same-sign and opposite-sign cases**.

``` c
int isLessOrEqual(int x, int y) {
  int _x = ~x + 1;
  int sub = y + _x;
  int sub_sign = sub >> 31 & 1;
  int xor_sign = x >> 31 & 1 ^ y >> 31 & 1;
  return !sub_sign & !xor_sign | x >> 31 & 1 & xor_sign;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c#L221-L227"><code>bits.c</code> L221–L227</a> (reformatted to fit)</p>

`_x` is `-x`, so `sub = y + _x` computes `y - x`; when the operands share a sign that subtraction cannot overflow, and `sub_sign` alone answers the question — 0 means `x ≤ y`. `xor_sign`, `x`'s sign bit XOR `y`'s, is 1 exactly when the signs differ, which is the case `sub` cannot handle. The return merges both: with `xor_sign = 0` the expression reduces to `!sub_sign`, and with `xor_sign = 1` it reduces to `x >> 31 & 1`, since a negative `x` against a non-negative `y` already guarantees `x ≤ y`.

### logicalNeg — implementing `!` without `!` { data-toc-label="logicalNeg" }

`!x` asks "is `x` zero?" The key insight: **for any number except 0, either it or its negation has its sign bit set**; only `0` and its negation are both non-negative.

``` c
int logicalNeg(int x) {
  int x_sign = x >> 31 & 1;
  int _x_sign = ~x + 1 >> 31 & 1;
  return ~(x_sign | _x_sign) << 31 >> 31 & 1;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c#L237-L241"><code>bits.c</code> L237–L241</a> (reformatted to fit)</p>

`x_sign` and `_x_sign` are the sign bits of `x` and `-x`. When `x = 0` both are 0; when `x != 0` at least one is 1, including the `Tmin` edge case, which stays negative under negation. So `x_sign | _x_sign` is 0 exactly when `x == 0`, and 1 otherwise; `~` inverts it, `<< 31 >> 31` smears the low bit across the word, and `& 1` leaves the logical negation.

### howManyBits — binary-searching the most significant bit { data-toc-label="howManyBits" }

Find the minimum number of bits to represent `x` in two's complement. This is Data Lab's finale, and its logic builds in layers.

``` c
int howManyBits(int x) {
  int standard_x = x >> 31 ^ x;
  int shift_16, shift_8, shift_4, shift_2, shift_1, shift;

  shift_16 = !!(standard_x >> 16) << 4;
  standard_x >>= shift_16;

  shift_8 = !!(standard_x >> 8) << 3;
  standard_x >>= shift_8;

  shift_4 = !!(standard_x >> 4) << 2;
  standard_x >>= shift_4;

  shift_2 = !!(standard_x >> 2) << 1;
  standard_x >>= shift_2;

  shift_1 = !!(standard_x >> 1) << 0;
  standard_x >>= shift_1;

  shift = shift_16 + shift_8 + shift_4 + shift_2 + shift_1;
  return shift + standard_x + 1;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c#L254-L275"><code>bits.c</code> L254–L275</a> (reformatted to fit)</p>

`standard_x = x >> 31 ^ x` normalizes the operand first: for `x ≥ 0` the shift contributes nothing and `standard_x` is `x`, while for `x < 0` it is `~x`. A negative number's width is set by its highest `0` bit, so inverting turns that into a highest `1` bit and unifies both signs into one question: where is the top set bit? The five `shift_*` steps binary-search it, asking "anything in the high 16 bits?" and, if so, recording weight 16 and shifting those bits away, then repeating for 8, 4, 2, and 1; each step uses `!!` to squash "nonzero" into `0/1` before `<<` turns it into a weight. Summing the weights gives the index of the top significant bit, with `standard_x` reduced to 0 or 1, and the `+1` pays for the sign bit.

!!! example "Why the `+1`"
    Two's complement always spends one bit on the sign. Take `howManyBits(12) = 5`: `12 = 0b01100`, whose top significant bit is 4 bits of magnitude — plus 1 sign bit = 5. Meanwhile `howManyBits(-1) = 1`, because `~(-1) = 0` needs only a single sign bit.

---

## Floating-Point Problems

Here the game isn't operator budgets but your grasp of IEEE 754's three fields — **sign `s`, exponent `exp`, fraction `frac`**. Single precision lays them out as `1 · 8 · 23` bits.

### floatScale2 — multiply a float by 2 { data-toc-label="floatScale2" }

In float-land, ×2 is usually just "exponent plus one" — but denormals and special values need care.

``` c
unsigned floatScale2(unsigned uf) {

  unsigned int exp = (uf << 1 & 0xFF << 24) >> 1;
  unsigned int frac = uf << 9 >> 9;
  unsigned int sign = uf & 0x1 << 31;

  if (exp == 0 && frac == 0 || exp == 0xFF << 23) { return uf; }

  if (exp == 0 && frac != 0) {
    frac <<= 1;
    return sign | exp | frac;
  }

  if (exp != 0) {
    exp += 0x1 << 23;
    if (exp == 0xFF << 23) {
      return sign | exp;
    }
    return sign | exp | frac;
  }

  return uf;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c#L288-L316"><code>bits.c</code> L288–L316</a> (reformatted to fit)</p>

Three lines carve out the exponent, fraction, and sign fields; the shifts do the masking. `exp == 0` with `frac == 0` is `±0`, and `exp == 0xFF << 23` is `±∞` or `NaN`; in both cases `2 * x` is the value itself, so the first branch returns `uf` untouched. With `exp == 0` and a nonzero fraction the number is denormal: shifting `frac` left by one doubles it, and if the top fraction bit carries it lands in the exponent field on its own, turning the value into a normal number with no separate fix-up. Otherwise the exponent is bumped, and if that saturates to `0xFF << 23` the code returns `sign | exp` and drops the fraction, producing infinity; if it does not, it reassembles `sign | exp | frac`.

### floatFloat2Int — float to integer { data-toc-label="floatFloat2Int" }

Equivalent to C's `(int) f`: compute the integer part of `1.frac × 2^shift`, where `shift` is the exponent with the bias removed.

``` c
int floatFloat2Int(unsigned uf) {

  int exp = uf >> 23 & 0xFF;
  int frac = uf << 9 >> 9;
  int sign = !!(uf & 0x1 << 31);
  int bias = 0x7F;
  int shift = exp - bias;

  if (shift < 0) { return 0; }

  if (shift > 30) {
    return 0x8u << 28;
  }

  frac += 0x1 << 23;

  if (shift <= 23) {
    frac >>= 23 - shift;
  } else {
    frac <<= shift - 23;
  }

  if (sign) {
    return ~frac + 1;
  } else {
    return frac;
  }
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c#L329-L366"><code>bits.c</code> L329–L366</a> (reformatted to fit)</p>

`exp` is the biased exponent field and `shift = exp - bias` is the true exponent. `shift < 0` means `|f| < 1`, so the integer part truncates to 0; `shift > 30` exceeds `int`'s range — including `∞` and `NaN` — and the convention is to return `0x80000000`. Otherwise `frac += 0x1 << 23` restores the leading 1 that IEEE 754 omits, making `frac` the full 24-bit mantissa. The mantissa still carries 23 fractional bits, so the binary point has to move: right-shift by `23 - shift` when `shift ≤ 23`, which throws the excess fraction away, and left-shift by `shift - 23` when `shift` is larger. `sign` then selects `~frac + 1` for negatives and `frac` for non-negatives.

### floatPower2 — compute 2.0^x { data-toc-label="floatPower2" }

Construct the bit pattern of `2^x` directly, branching on whether `x` lands in the normal or denormal range.

``` c
unsigned floatPower2(int x) {

  int exp, frac;
  int bias = 0x7F;

  if (x > 127) {
    return 0xFF << 23;
  }

  if (x < -149) { return 0; }

  if (x >= -126) {
    exp = bias + x;
    frac = 0;
  } else {
    exp = 0x0;
    frac = 0x1 << (149 + x);
  }

  return exp << 23 | frac;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Data_Lab/datalab-handout/bits.c#L380-L407"><code>bits.c</code> L380–L407</a> (reformatted to fit)</p>

Two range checks come first: `x > 127` exceeds the largest normal exponent and returns `0xFF << 23`, that is `+∞`; `x < -149` is below the smallest denormal and returns 0. In between, `x >= -126` is the normal range, where `2^x` has a zero fraction and an exponent field of `bias + x`. The remaining window, `-149 <= x <= -127`, is denormal, so `exp` stays 0 and the single fraction bit sits at `149 + x`: position 0 for `2^-149`, the smallest denormal. The final `exp << 23 | frac` assembles both cases.
