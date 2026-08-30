# Fast unicode (UTF-8) validation

TLDR: This article describes a very fast algorithm for UTF-8 unicode validation. It has two paths, ascii and UTF-8, both paths are autovectorized.

## Introduction

This UTF8 validator is based on a branchless lookbehind algorithm which is autovectorizable. It checks every byte position from a 4-byte window (current byte + 3-byte lookbehind), and it has an ascii check fast path that skips ascii text blocks.

The ASCII skip algorithm is an adaptation of the one in Lemire's [Performance trick : optimistic vs pessimistic checks](https://lemire.me/blog/2025/12/20/performance-trick-optimistic-vs-pessimistic-checks/) article.

The algorithm is derived from the UTF-8 specification table and the classical branchy implementation.

A prettified version of the algorithm (with "named expressions") can be found in [its repository](https://github.com/nitely/nim-utf8-validator/blob/master/utf8_validator.nim). It's implemented in [Nim](https://nim-lang.org/).

## Benchmarks

```text
data/ascii.txt  --  0.57 MiB, 0.00% non-ascii
  validator      verdict   best GB/s   median   best ms
  -----------------------------------------------------
  nitely           valid       82.91    81.09     0.007
  branchy          valid        2.04     1.94     0.294
  dfa              valid        0.57     0.57     1.049

data/jp_random.txt  --  0.02 MiB, 98.36% non-ascii
  validator      verdict   best GB/s   median   best ms
  -----------------------------------------------------
  nitely           valid        9.85     9.71     0.003
  branchy          valid        1.97     1.94     0.013
  dfa              valid        0.55     0.54     0.048

data/hongkong.html  --  1.72 MiB, 8.37% non-ascii
  validator      verdict   best GB/s   median   best ms
  -----------------------------------------------------
  nitely           valid       17.35    17.08     0.104
  branchy          valid        1.88     1.79     0.962
  dfa              valid        0.57     0.54     3.171

data/twitter.json  --  0.60 MiB, 15.11% non-ascii
  validator      verdict   best GB/s   median   best ms
  -----------------------------------------------------
  nitely           valid       21.12    20.11     0.030
  branchy          valid        1.89     1.87     0.335
  dfa              valid        0.57     0.57     1.107
```

Validators:
- `nitely`: the validator explained in this article.
- `branchy`: classical branchy validator derived from the UTF8 spec.
- `dfa`: a [DFA based validator](https://bjoern.hoehrmann.de/utf-8/decoder/dfa/). 

The algorithm is quite fast for ASCII only text, so the more ASCII in the text the faster it gets. The `jp_random` text contains almost pure UTF-8 sequences, and so it should be taken as the closest to base performance.

Note the `branchy` algorithm is quite fast thanks to CPU branch prediction. In pathological cases (not included here) it can be slower than the DFA algorithm. It's also capable of bailing out sooner on invalid UTF-8, however these benchmarks contain valid UTF-8.

## Correctness proof

### 1. Assumption

Standard UTF-8 well-formedness table:

| Code points | Byte 1 | Byte 2 | Byte 3 | Byte 4 |
|---|---|---|---|---|
| U+0000..U+007F | `00`..`7F` | | | |
| U+0080..U+07FF | `C2`..`DF` | `80`..`BF` | | |
| U+0800..U+0FFF | `E0` | `A0`..`BF` | `80`..`BF` | |
| U+1000..U+CFFF | `E1`..`EC` | `80`..`BF` | `80`..`BF` | |
| U+D000..U+D7FF | `ED` | `80`..`9F` | `80`..`BF` | |
| U+E000..U+FFFF | `EE`..`EF` | `80`..`BF` | `80`..`BF` | |
| U+10000..U+3FFFF | `F0` | `90`..`BF` | `80`..`BF` | `80`..`BF` |
| U+40000..U+FFFFF | `F1`..`F3` | `80`..`BF` | `80`..`BF` | `80`..`BF` |
| U+100000..U+10FFFF | `F4` | `80`..`8F` | `80`..`BF` | `80`..`BF` |

The `branchy` algorithm implementation can be found [here](https://github.com/nitely/nim-utf8-validator/blob/8ce234ac41652f92f1e2b89a0d4cdd4db0e3ac23/bench/utf8_branchy.nim).

The table and `branchy` algorithm are assumed correct. The equivalence can be checked in three parts:

- **Length**: which bytes must be continuation bytes; continuation bytes missing or unrequired.
- **Byte 2**: range limits after `E0`, `ED`, `F0`, `F4`; rejection of `C0`, `C1`, `F5..FF`.
- **Bounds and truncation**.

---

### 2. Character boundaries

This algorithm checks each position independently doing a three-byte lookbehind.

```nim
template mustBeContinuation(prev1, prev2, prev3: uint8): uint8 =
  uint8(prev1 >= 0xC0'u8) or uint8(prev2 >= 0xE0'u8) or uint8(prev3 >= 0xF0'u8)
```

| Test | `p[i]` must be a continuation because |
|---|---|
| `p[i-1] >= C0` | `p[i-1]` could lead a character of 2 bytes or more |
| `p[i-2] >= E0` | `p[i-2]` could lead a character of 3 bytes or more |
| `p[i-3] >= F0` | `p[i-3]` could lead a character of 4 bytes |

For example: `C0`, `C1`, `F5..FF` pass these thresholds but never start a character. §3 rejects them.

These cases cannot overlap for valid UTF-8. A position belongs to one character.

```nim
let isCont = uint8(uint8(p[i]) >= 0x80'u8) and uint8(uint8(p[i]) < 0xC0'u8)
```

```nim
template checkMultibyteLengths(isCont, prev1, prev2, prev3: uint8): uint8 =
  mustBeContinuation(prev1, prev2, prev3) xor isCont
```

The result is non-zero in either of these cases:

1. a continuation is required and absent.
2. a continuation is present and not required.

Zero at every position reproduces the `branchy` branch boundaries: after a lead of length `L`, positions `+1 .. +L-1` are required continuations and `+L` is not, so `+L` starts the next character.

Two of the tests are true at the same position only when a byte `>= C0` sits where an earlier lead already required a continuation. That byte is not a continuation, so the check at its own position has already failed.

`F0 C2 80`: at position 2 both `p[1] >= C0` and `p[0] >= E0` hold, but position 1 has already failed; ie: `F0` requires a continuation there and `C2` is not one.

---

### 3. Byte 2

Bytes 3 and 4 are `80..BF` in every row. All restrictions beyond continuation status fall on byte 2.

| Lead | Required byte 2 | Error when |
|---|---|---|
| `C0`, `C1` | never starts a character | always |
| `C2`..`DF` | `80..BF` | never |
| `E0` | `A0..BF` | `input < A0` |
| `E1`..`EC` | `80..BF` | never |
| `ED` | `80..9F` | `input >= A0` |
| `EE`, `EF` | `80..BF` | never |
| `F0` | `90..BF` | `input < 90` |
| `F1`..`F3` | `80..BF` | never |
| `F4` | `80..8F` | `input >= 90` |
| `F5`..`FF` | never starts a character | always |

```nim
template checkSpecialCases(input, prev1, isCont: uint8): uint8 =
  isCont and (uint8((prev1 and 0xFE'u8) == 0xC0'u8) or
    uint8(prev1 == 0xE0'u8) and (uint8(input >= 0xA0'u8) xor 1'u8) or
    uint8(prev1 == 0xED'u8) and uint8(input >= 0xA0'u8) or
    uint8(prev1 == 0xF0'u8) and (uint8(input >= 0x90'u8) xor 1'u8) or
    uint8(prev1 == 0xF4'u8) and uint8(input >= 0x90'u8) or
    uint8(prev1 >= 0xF5'u8))
```

`(prev1 and 0xFE) == 0xC0` matches exactly `C0` and `C1`.

The `isCont` guard does not hide any invalid case. For `prev1` in `C0`, `C1`, `F5..FF`:

| `p[i]` | Caught by |
|---|---|
| a continuation | `checkSpecialCases` |
| not a continuation | `checkMultibyteLengths` — the lead requires one |
| absent (end of input) | the check at `n`, §4 |

At bytes 3 and 4 `checkSpecialCases` is always zero: `prev1` is a continuation byte there, and `80..BF` is none of `C0`, `C1`, `E0`, `ED`, `F0`, `F4` and is below `F5`. The table constrains bytes 3 and 4 only to `80..BF`, which §2 covers.

---

### 4. Beginning and end

Positions `0`, `1`, `2` have fewer than three preceding bytes.

```nim
template checkBounded(p: openArray[char], n, i: int): uint8 =
  var window: array[4, char]  # zeroed
  for k in 0 .. 3:
    template idx: untyped = i - k
    if idx >= 0 and idx < n:
      window[3 - k] = p[idx]
  checkUtf8Bytes(window, 3)
```

Zero is neither a continuation byte nor a lead byte.

End of input: since an incomplete character lacks a final byte to trigger an error, the validator runs an extra check at index `n` and uses a zero in place of the absent byte.

```nim
template isIncomplete(p: openArray[char], n: int): uint8 =
  checkBounded(p, n, n)
```

At `i = n`, `isCont` is 0, which bypasses the byte-2 checks and leaves:

```text
p[n-1] >= C0 or p[n-2] >= E0 or p[n-3] >= F0
```

| Input ends with | Missing byte |
|---|---|
| a lead byte (`C2`, `E0`, `F0`, ...) | second |
| a 3- or 4-byte lead plus one continuation | third |
| a 4-byte lead plus two continuations | fourth |

Trailing `C0`, `C1`, `F5..FF` are `>= C0` and are caught here.

Checking `n` is enough: `n+1` would test `p[n-1] >= E0` and `p[n-2] >= F0`, `n+2` would test `p[n-1] >= F0`, and each is implied by a test already made at `n`.

---

### 5. Combined check

```nim
template checkUtf8Bytes(p: openArray[char], i: int): uint8 =
  checkSpecialCases(uint8(p[i]), uint8(p[i - 1]), isCont) or
    checkMultibyteLengths(isCont, uint8(p[i - 1]), uint8(p[i - 2]), uint8(p[i - 3]))
```

For each position `0 .. n-1`, the check catches:

- a continuation byte where none is required.
- a missing continuation byte.
- `C0` and `C1`.
- invalid byte 2 after `E0`, `ED`, `F0`, `F4`.
- `F5..FF`.

At `n`: a character left incomplete.

The byte sequence is valid *iff* the error value is zero at every position `0 .. n-1` and at `n`.

Fixed four-byte window, no carried character index or loop state: the error values combine with bitwise `or` independent of evaluation order.

---

### 6. `validateUtf8` and the ASCII fast path

```nim
func validateUtf8*(p: openArray[char]): bool =
  var error = 0'u8
  let n = p.len
  let prefix = min(n, 3)
  for i in 0 ..< prefix:
    error = error or checkBounded(p, n, i)
  var i = prefix
  while i + utf8Block <= n:
    if not isAscii(p, i):
      for j in 0 ..< utf8Block:
        error = error or checkUtf8Bytes(p, i + j)
    i += utf8Block
  while i < n:
    error = error or checkUtf8Bytes(p, i)
    inc i
  error = error or isIncomplete(p, n)
  error == 0'u8
```

The first loop covers `0..2`, the only positions whose lookbehind reaches before the input, hence `checkBounded`; loop 2 takes whole blocks; loop 3 takes the remainder; `isIncomplete` takes `n`.

```nim
template isAscii(p: openArray[char], i: int): bool =
  var res = 0'u8
  for j in 0 ..< utf8Block + 3:
    res = res or uint8(p[i - 3 + j])
  res <= 0x7F'u8
```

With `B = utf8Block`, the checks at `i .. i+B-1` read:

```text
i        -> p[i-3 .. i]
i+1      -> p[i-2 .. i+1]
i+2      -> p[i-1 .. i+2]
i+3      -> p[i   .. i+3]
...
i+B-1    -> p[i+B-4 .. i+B-1]
```

The two loops cover `p[i-3 .. i+B-1]`, `B+3` bytes; ie: the range `isAscii` scans. Only the first three positions of a block read below `i`.

All ASCII in that range: no continuation bytes, no lead bytes, so every skipped check would return zero.

The three-byte margin is required. With `utf8Block = 256` blocks start at 3, 259, 515; take:

```text
258: C2
259: A
```

The error is at position 259; ie: `C2` requires a continuation, `A` is not one. Scanning only `259..514` finds pure ASCII, skips the block, and accepts invalid input. Scanning from `256` sees the `C2` and blocks the skip.

For characters spanning block boundaries, all bytes relevant to a skipped check are either in the block itself or the preceding three bytes.

---

### 7. Result

Under §1, this [utf8_validator](https://github.com/nitely/nim-utf8-validator/blob/master/utf8_validator.nim) accepts the same strings as `branchy`.

| `branchy` | `utf8_validator` |
|---|---|
| continuation-byte requirements, `i += L` | `mustBeContinuation` xor `isCont` (§2) |
| byte-2 ranges; `C0`, `C1`, `F5..FF` | `checkSpecialCases` (§3) |
| `i+L >= n` truncation | the check at `n` (§4) |

This checks every position (§6), and blocks are skipped only when every byte the skipped checks would read is ASCII.

## Notes

- Both the ASCII path and the UTF8 path get autovectorized.
- Why is the utf8 block size 256? It seems like the best value for the benchmarks. But in general it is likely better to set it to 64 or 128; it will make the pure ascii check slower, but 256 bytes windows of pure ascii are likely much more rare than 64 bytes. Aside from the ascii benchmark, a block of +64 didn't make a drastic difference in my machine. For the record 64 halves the ascii bench performance for me. Anything lower than 64 will hurt perf, likely because of CPU cache line size.

I hope you enjoyed this article and found it useful. Until next time.
