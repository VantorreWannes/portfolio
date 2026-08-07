---
icon: material/file-compare
---

# cos_lcs_zig

**A zero-allocation, iterator-based greedy approximation of the Longest Common Subsequence, written in Zig.**

[:octicons-mark-github-16: Source Repository](https://github.com/VantorreWannes/cos_lcs-zig){ .md-button .md-button--primary }
[:octicons-book-24: Prior art: Fraser (1995)](https://theses.gla.ac.uk/74575/1/10992195.pdf){ .md-button }


---

!!! abstract "Project overview"
    `cos_lcs_zig` implements a greedy LCS approximation as a lazy iterator over two byte slices. At each step it emits the matching pair (i, j) that **minimises i + j** - the earliest match on the lowest anti-diagonal - then advances both cursors past it. It never allocates, keeps O(1) state (≈ 2 KiB on the stack), and costs O(n + m) per emitted item.

    The rule turned out to be an independent rediscovery: it is a close variant of **Best-Next**, a greedy LCS approximation analysed in C. B. Fraser's 1995 PhD thesis *Subsequences and Supersequences of Strings* - which also shows why no algorithm in this family can carry a good worst-case guarantee.[^fraser]

[^fraser]: C. B. Fraser, *Subsequences and Supersequences of Strings*, PhD thesis, University of Glasgow, 1995. [PDF](https://theses.gla.ac.uk/74575/1/10992195.pdf). Best-Next is §5.3; Chapter 5 analyses the worst-case behaviour of greedy LCS/SCS approximations generally.

## Background

The LCS problem is easy for two strings and brutally hard in general:

| Setting | Status |
| --- | --- |
| k = 2 strings | Polynomial: classic DP in O(nm) time (Wagner–Fischer 1974); many refinements exist |
| k unbounded | NP-complete, even over a binary alphabet (Maier 1978) |
| Approximation, general k | No polynomial-time guarantee of the form k^δ^ (δ > 0) unless P = NP (Jiang & Li); MAX SNP-hard even on a binary alphabet |

So for two strings the interesting question is not exactness but **cost**: a cheap, streaming, allocation-free heuristic that gets close to optimal on realistic inputs. That is the niche `cos_lcs_zig` occupies.

## How it works

```text
remaining source ──scan──▶ candidate pair (i, j)
                             j = first occurrence of source[i]
                                in the remaining target
                             │
                             └─ pick the pair minimising i + j,
                                emit it, advance both cursors past it,
                                repeat until no common byte remains
```

Two pruning rules keep each step cheap:

- `sum == 0` → stop scanning immediately. A match at (0, 0) can never be beaten.
- `s_idx >= min_sum` → stop scanning. Every later source index only makes the sum larger, so no better candidate exists further along.

### Worked example

Iterating `source = [2, 1, 0, 3]` against `target = [0, 1, 2, 3]` (from the unit tests):

| Step | Remaining source | Remaining target | Candidates (i + j) | Emitted |
| ---: | --- | --- | --- | --- |
| 1 | `2 1 0 3` | `0 1 2 3` | `2`: 0+2, `1`: 1+1, `0`: 2+0, `3`: 3+3 | `2` (three-way tie at 2 → earliest source index wins) |
| 2 | `1 0 3` | `3` | `3`: 2+0 | `3` |
| 3 | - | - | none | *(done)* |

Output: `2 3` - which happens to be optimal here (LCS length 2). See the [analysis](#analysis) for when it isn't.

## API reference

The whole library is one type, `CosLcsIterator` (re-exported from `src/root.zig`):

| Item | Signature | Notes |
| --- | --- | --- |
| `init` | `fn init(source: []const u8, target: []const u8) CosLcsIterator` | O(1), no allocation |
| `next` | `fn next(self: *) ?u8` | Next common byte, `null` when exhausted |
| `nextPair` | `fn nextPair(self: *) ?Pair` | Same, with original-slice indices (for alignment/diff use) |
| `reset` | `fn reset(self: *) void` | Rewind both cursors and iterate again |
| `Pair` | `struct { item: u8, source_index: usize, target_index: usize }` | Indices are absolute, into the *original* slices |
| `Cursor` | `struct { source_index: usize, target_index: usize }` | Current iterator position |

```zig
const std = @import("std");
const CosLcsIterator = @import("cos_lcs_zig").CosLcsIterator;

pub fn main() void {
    var it = CosLcsIterator.init(&[_]u8{ 2, 1, 0, 3 }, &[_]u8{ 0, 1, 2, 3 });
    while (it.nextPair()) |pair| {
        std.debug.print("{d} at source[{d}], target[{d}]\n", .{
            pair.item, pair.source_index, pair.target_index,
        });
    }
    // 2 at source[0], target[2]
    // 3 at source[3], target[3]
}
```

Because results are produced lazily, memory use is independent of the output length, and callers can stop early - e.g. for threshold questions like "is there a common subsequence of length ≥ k?"

## Prior art: Fraser (1995)

The iterator's rule sits in the greedy left-to-right family whose worst-case behaviour was characterised in Fraser's thesis (Chapter 5):

| Algorithm | Greedy rule | Worst case (tight) |
| --- | --- | --- |
| `cos_lcs_zig` (this repo) | match minimising i + j | up to ≈ n/2 off optimal, **even for k = 2** (see below) |
| Best-Next (§5.3) | symbol maximising the shortest remaining suffix (≡ minimising max(i, j) for equal lengths) | l/b ≤ n/2, tight for k ≥ 3, binary alphabet |
| Long-Run (Jiang & Li; §5.2) | longest single-symbol run σⁱ common to all strings | l/r ≤ z (z = alphabet size) |

The two rules look interchangeable but are not: the counterexample in the [analysis](#approximation-quality) is one where they pick different first matches, and Best-Next finds the optimum while the min-sum rule does not.

## Analysis

### Complexity

| Quantity | Cost |
| --- | --- |
| `init` / `reset` | O(1) |
| One `nextPair` step | O(n + m) - rebuild the 256-entry first-occurrence table over the remaining target, then scan the remaining source (with early exits) |
| Full iteration | O(l·(n + m)), where l = emitted pairs ≤ min(n, m); worst case O(nm) |
| Extra memory | O(1) - fixed 256-slot table; the iterator is ≈ 2 KiB, stack-resident, zero heap |

Note the honest worst case: O(nm) is the *same* asymptotic class as exact DP. What the iterator buys you is tiny constants, zero allocation, laziness, and early exit - not a better complexity class.

### Approximation quality

The README states an imperical approximation factor of 0.8 (observed on random inputs), not a proven guarantee.

=== "Worst case"

    Take `source = "z123abc"`, `target = "456abcz"` (digits are junk bytes appearing in one string only):

    - Candidates: `z` at (0, 6), sum 6; `a` at (4, 3), sum 7; `b`, `c` larger.
    - The min-sum rule takes `z` - and the remaining target is empty. **Output length: 1.**
    - The LCS is `"abc"`, length **3**.

    Generalising the junk/block padding gives instances with LCS/greedy ≈ n/2. Best-Next, for comparison, picks `a` here (max remaining-suffix rule) and goes on to find `"abc"` - proof the two variants genuinely differ.

=== "Random inputs"

    For context when reading benchmark output: the limiting expected ratio
    **F(z, 2) = limₙ→∞ E[|LCS|] / n** for two random strings has been studied since Chvátal–Sankoff. Estimates from Fraser's experiments (n = 100,000, three instances per z), with the best theoretical bounds known at the time:

    | z | Lower bound | Estimate of F(z, 2) | Upper bound |
    | ---: | ---: | ---: | ---: |
    | 2 | 0.7615 | 0.8123 | 0.8376 |
    | 4 | 0.5454 | 0.6542 | 0.7297 |
    | 8 | 0.4223 | 0.5151 | 0.5967 |
    | 16 | - | 0.3960 | - |

    These are the natural denominators when expressing the iterator's output as a fraction of optimal on the random corpora used by the benchmark suite (whose alphabet sizes 2, 4, 16 are covered above). Caveat: the suite uses n ≤ 2000, so finite-size effects apply; the estimates come from n = 100,000.

## Benchmarking

A [zBench](https://github.com/hendriknielaender/zBench) suite is included. It instantiates one benchmark per (length, alphabet) pair - 56 in total - over uniform random bytes:

| Parameter | Values |
| --- | --- |
| Length n = m | 10, 100, 250, 500, 1000, 1500, 2000 |
| Alphabet size z | 1, 2, 4, 16, 32, 64, 128, 255 |

```bash
zig build bench
```

Reading the results:

- z = 1 is the **stress case**, not the easy one: every byte matches, so l = n and each step rescans the remainder - the realised O(n²) worst case.
- Large z **flatters the algorithm twice**: few matches means both a short output *and* short scans per step.
- The suite measures throughput only.

## Development

Requires Zig `0.15.0-dev.919+044ccf413` or compatible (per `build.zig.zon`).

```bash
zig build test    # unit tests (next / nextPair / reset)
zig build bench   # benchmark suite
zig build docs    # emit API docs into zig-out/docs
python -m http.server 8000 -d zig-out/docs/   # browse them
```

```text
src/cos.zig    CosLcsIterator - the entire algorithm
src/root.zig   library entry point, re-exports CosLcsIterator
src/bench.zig  zBench suite
src/main.zig   placeholder binary (the library is the product)
```

*[LCS]: Longest Common Subsequence
*[SCS]: Shortest Common Supersequence
*[DP]: Dynamic Programming