---
icon: material/label
title: L&P - Universal Language from 4 Pulse Lengths
description: A universal language for alien contact built from four pulse lengths, deriving combinators, booleans, and Church arithmetic without 0 or 1.
image: assets/lap-linkedin.png
---

# L&P

A universal language for alien contact. Four pulse lengths: no alphabet, no numbers, no convention for `1` or `0`.

`111 0 111 0 11 0 1 0 1` is `..|??`: two unknown bits into NAND.

## Signal format

Pulses are separated by a 1-unit gap (`0`).

| Pulse | Symbol | Meaning |
| --- | --- | --- |
| 1 unit (`1`) | `?` | Unknown bit (0 or 1 until observed) |
| 2 units (`11`) | <code>&#124;</code> | NAND |
| 3 units (`111`) | `.` | Binary application (prefix order) |
| 4 units (`1111`) | `!` | Iota combinator (`! x = x S K`) |

## Uncertainty and truth

Each `?` is a distinct unknown bit. Copies made by combinators stay linked to the same bit.

Evaluating an expression tests every `0`/`1` assignment of its distinct `?`s:

- Always `1`: `true` (`K`, `lambda x y. x`)
- Always `0`: `false` (`K I`, `lambda x y. y`)
- Mixed: `true_count / total_assignments` (`?` is `1/2`; `..|??` is `3/4`)

Church numeral `0` (`K I`) shares the normal form of bit `0`. Church numeral `1` is `I` (`lambda f x. f x`), not `K`.

Read `|` as NOR instead of NAND: `0` and `1` swap labels (`K` and `K I`), and every circuit and theorem still holds.

## Transmission

Sent in order, from `?` to arithmetic:

```text
# Stage 1: Uncertainty and NAND
?
..|??

# Stage 2: Combinators from Iota (! x = x S K)
I = .!!            # ! ! = ! S K = S K (K K) = I
K = .!.!I          # ! (! I) = ! (S K) = S K S K = K
S = .!K            # ! K = K S K = S
B = ..S.K S K
C = ..S..B B S.K K

# Stage 3: Deriving 1 and 0 without primitives
NOT = ..S|I
TAUTOLOGY = ..S|NOT
ONE = .TAUTOLOGY?  # 0|(0|0) = 1, 1|(1|1) = 1
ZERO = ..|ONE ONE  # 1|1 = 0

ONE
ZERO
.NOT ONE
.NOT ZERO

# Stage 4: Logic gates and exact probabilities
AND = ..S.K.B NOT|
OR = ..B..C B NOT..B|NOT

..AND ONE ONE
..AND ONE ZERO
..AND??
..OR??

# Stage 5: Causality and tautologies
LEM = ..S OR NOT

..|ZERO?
..|ONE?
.LEM?

# Stage 6: Church numerals and arithmetic
N0 = .K I
N1 = I
SUCC = .S B
N2 = .SUCC N1
N3 = .SUCC N2
ADD = ..C I SUCC
MUL = B
IS_ZERO = ..C..C I.K ZERO ONE

N1
N2
N3
..ADD N2 N3
..MUL N2 N3
.IS_ZERO N0
.IS_ZERO N3
```
