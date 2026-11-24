---
icon: material/label
---

# L&P

A prefix-order combinator and bit-logic language built from 4 pulse lengths.

## Signal format

Tokens are pulses separated by a 1-unit gap of silence. A 3+ unit gap of silence ends a message.

| Pulse | Symbol | Meaning |
| --- | --- | --- |
| 1 unit | `?` | Unknown bit (0 or 1 until observed) |
| 2 units | <code>&#124;</code> | NAND |
| 3 units | `.` | Binary application (prefix order) |
| 4 units | `!` | Iota combinator (`! x = x S K`) |

## Evaluation

Each `?` in an expression is a distinct unknown bit. Copies of a `?` made by combinators stay linked to that same bit.

To evaluate an expression, test every 0/1 assignment of its `?`s:

- Always 1: true
- Always 0: false
- Mixed: uncertain (`true_count / total_assignments`)

## Representation and duality

- `x | (x | x)` on `?` reduces to `1` (`K`), and `1 | 1` reduces to `0` (`K I`).
- Bits are Church booleans: `1` is `K` (`lambda x y. x`) and `0` is `K I` (`lambda x y. y`).
- Church numeral `0` (`K I`) shares the normal form of bit `0`. Church numeral `1` is `I` (`lambda f x. f x`), not `K`.
- If `|` is read as NOR instead of NAND, `0` and `1` swap labels and every circuit and theorem still holds.

## Stages

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
