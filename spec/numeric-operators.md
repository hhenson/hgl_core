# Numeric operators and policies

Status: agreed baseline, explicit operator configuration, approximate equality,
full comparison, and IEEE division default. The error-policy API illustration
below remains proposed. No executable conformance or native ABI is specified.

## Contract

The [language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/expressions-and-bindings.md)
owns symbol syntax, configuration binding, precedence, and phase checking.
Core owns numerical domains, public configuration, results, and errors.

- **NUM-1.** For i64 operands, `+`, `-`, `*`, `//`, and `%` return i64.
  Mixed i64/f64 and f64 arithmetic returns f64; `/` always returns f64.
  Unary negation preserves the numeric type. Integer floor division rounds
  toward negative infinity; modulo has the divisor's sign when nonzero.
- **NUM-2.** An unrepresentable i64 result produces an HGL error; it neither
  wraps nor silently promotes. Floating arithmetic rounds to binary64 and
  preserves IEEE special values, including signed zero and overflow infinity.
  Mixed numeric comparison must not first round an i64 operand to f64 and
  thereby lose a distinction between the operand values.
- **NUM-3.** True division `/`, whose numeric result is f64, defaults to the
  `ieee` zero policy:
  a finite nonzero numerator divided by signed zero produces signed infinity;
  zero divided by zero produces NaN. The infinity sign combines both operand
  signs. The `nan` zero policy instead produces NaN for every zero divisor.
  An explicit error-producing alternative must also be available. The zero
  option applies to zero divisors, not to unrelated operand errors. Policies
  requiring NaN/infinity need a result type that can represent those values.
- **NUM-4.** Plain numeric `==` is exact and `!=` is its complement. Configured
  `==` accepts absolute (`abs`) and relative (`rel`) tolerance, each defaulting
  to zero. For finite operands, closeness is the symmetric condition below.
  Tolerances must be finite and nonnegative. Configured `!=` is the complement
  under the same options. An invalid tolerance is an HGL error under the
  ordinary phase rules.

```text
abs(a - b) <= max(abs_tolerance,
                 rel_tolerance * max(abs(a), abs(b)))
```

- **NUM-5.** NaN is neither equal nor close to any value, including itself.
  Infinities are equal/close only to the same signed infinity; opposite signed
  zeros are equal/close. Apply these cases before the finite closeness rule.
  The finite test must not admit an incorrect result through overflow of
  intermediate subtraction or scaling.
- **NUM-6.** The numeric full-comparison operator `compare_`, selected by
  `<=>`, returns an ordinary comparison outcome: `less`, `equivalent`,
  `greater`, or `unordered`. A NaN operand yields `unordered`; equivalent
  numeric values, including opposite signed zeros, yield `equivalent`.
  Numeric Boolean comparisons obey the table below. An outcome has no implicit
  bool or numeric conversion.
- **NUM-7.** Approximate closeness is not a transitive equivalence relation.
  A tolerance option grants no ordering law, dictionary-key identity, or
  optimizer permission. Sorting a domain admitting unordered values requires
  an explicit total-order policy defining NaN placement/equivalence and
  signed-zero treatment. Approximate equality does not configure `<=>`.
- **NUM-8.** A configuration selects the same nominal operator and applicable
  contract. Omitted options use its public defaults; imports, target choice,
  and optional folding must not silently alter them. Value and temporal
  execution preserve the same numerical outcomes under their existing phase,
  lifting, validity, publication, and error rules. A NaN/infinity publication
  is present data. Required-constant errors and optional folding follow the
  language's error-phase contract.

| Numeric expression | Full-comparison outcomes producing true |
| --- | --- |
| `a == b` | `equivalent` |
| `a != b` | `less`, `greater`, `unordered` |
| `a < b` | `less` |
| `a <= b` | `less`, `equivalent` |
| `a > b` | `greater` |
| `a >= b` | `greater`, `equivalent` |

## Examples and conformance observations

Accepted expression fragments; the policy names `ieee` and `nan` denote core
configuration values in scope, not new numeric literals:

```hgl
1.0 / 0.0
1.0 /{zero: ieee} 0.0
1.0 /{zero: nan} 0.0
1.0 =={abs: 0.01} 1.005
1.0 <=> 2.0
```

Core owns the following expected observations. Executable cases are planned.
NaN/infinity names in the table describe outcomes, not source literal syntax.

| Case | Expected observation |
| --- | --- |
| `7 / 2`; `-7 // 3`; `-7 % 3` | `3.5`; `-3`; `2` |
| Maximum i64 plus one; negation of minimum i64 | HGL errors, without wrapping |
| `9007199254740993 == 9007199254740992.0` | false |
| `1.0 / 0.0`; `1.0 / -0.0`; `0.0 / 0.0` | Positive infinity; negative infinity; NaN |
| `1.0 /{zero: nan} 0.0` | NaN |
| `1.0 == 1.005`; `1.0 =={abs: 0.01} 1.005` | false; true |
| Relative-only tolerance comparing a nonzero value to zero, with rel less than one | false |
| NaN compared with itself by `<=>`, `==`, and `!=` | unordered; false; true |
| Opposite signed zeros compared by `<=>` | equivalent |
| Same signed infinities compared with a tolerance | true |
| `1e308 =={rel: 1.9} -1e308` | false, despite overflow in a naive difference/threshold calculation |
| Tolerance 1 applied to 0/0.75, 0.75/1.5, and 0/1.5 | true; true; false |

## Proposed error-policy API

```hgl
a /{zero: error} b
```

This proposed spelling selects an explicit HGL error outcome for a zero divisor.
It introduces no host-exception requirement. Its public declaration, result/error
carrier, and codes have not been fixed; this example makes no signature or
executable conformance claim.
