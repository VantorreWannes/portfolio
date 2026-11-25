---
icon: material/label
---

# L&P

A universal language for alien contact. Four pulse lengths: no alphabet, no numbers, no convention for `1` or `0`.

```text
111 0 111 0 11 0 1 0 1 000
```

`111 0 111 0 11 0 1 0 1 000` is `..|??`: two unknown bits into NAND.

## Signal format

Pulses are separated by a 1-unit gap (`0`). A 3+ unit gap (`000`) ends a message.

| Pulse | Symbol | Meaning |
| --- | --- | --- |
| 1 unit (`1`) | `?` | Unknown bit (0 or 1 until observed) |
| 2 units (`11`) | <code>&#124;</code> | NAND |
| 3 units (`111`) | `.` | Binary application (prefix order) |
| 4 units (`1111`) | `!` | Iota combinator (`! x = x S K`) |

## Bootstrapping combinators

From `! x = x S K`, every combinator reduces from pure pulses:

```text
I = .!!          # ! ! = ! S K = S K (K K) = I
K = .!.!I        # ! (! I) = ! (I S K) = ! (S K) = S K S K = K
S = .!K          # ! K = K S K = S
B = ..S.K S K
C = ..S..B B S.K K
```

## Uncertainty and truth

Each `?` is a distinct unknown bit. Copies made by combinators stay linked to the same bit.

| Expression | Assignments (`0`/`1`) | Result |
| --- | --- | --- |
| `?` | `0`, `1` | `1/2` (uncertain) |
| `..|??` | `(0,0)->1`, `(0,1)->1`, `(1,0)->1`, `(1,1)->0` | `3/4` (uncertain) |
| `.TAUTOLOGY?` | `0 -> 0|(0|0) = 1`, `1 -> 1|(1|1) = 1` | `1` (`K`, `lambda x y. x`) |
| `..|ONE ONE` | `1 | 1 = 0` | `0` (`K I`, `lambda x y. y`) |

Church numeral `0` (`K I`) shares the normal form of bit `0`. Church numeral `1` is `I` (`lambda f x. f x`), not `K`.

Read `|` as NOR instead of NAND: `0` and `1` swap labels (`K` and `K I`), and every circuit and theorem still holds.

## Transmission

Sent in order, from `?` to arithmetic:

```text
# Stage 1: Uncertainty and NAND
?
..|??

# Stage 2: Combinators from Iota
I = .!!
K = .!.!I
S = .!K
B = ..S.K S K
C = ..S..B B S.K K

# Stage 3: Deriving 1 and 0 without primitives
NOT = ..S|I
TAUTOLOGY = ..S|NOT
ONE = .TAUTOLOGY?
ZERO = ..|ONE ONE

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
