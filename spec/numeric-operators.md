# Numeric operators and policies

Status: agreed baseline, explicit operator configuration, approximate equality,
full comparison, IEEE division default, and checked-division result contract.
No executable conformance or native ABI is specified.

## Contract

The [language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/expressions-and-bindings.md)
owns symbol syntax, configuration binding, precedence, and phase checking.
Core owns numerical domains, public configuration, results, and errors.

- **NUM-1.** For i64 operands, `+`, `-`, `*`, `//`, and `%` return i64.
  Unconfigured mixed i64/f64 and f64 arithmetic returns f64; `/` defaults to f64.
  NUM-9 defines the configured division result types.
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
  The `error` alternative follows NUM-9. The zero
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
- **NUM-9.** Division with `zero: ieee` or `zero: nan` returns f64; division
  with `zero: error` returns `result<f64, error>`. Resolve the zero policy during
  source checking because it determines the result type. For the error policy,
  either signed zero divisor produces `err(...)` carrying error code
  `numeric.division_by_zero`; every nonzero divisor produces `ok(value)` from
  ordinary IEEE division. This includes successful NaN/infinity results where
  IEEE division produces them. This policy checks only the divisor and does
  not catch unrelated errors evaluating the operands.
- **NUM-10.** A checked-division Err is ordinary result data. It neither
  implicitly fails a node nor requires an exception. Temporal results publish
  atomically under the existing admission rules; an Err is a present publication
  and does not prevent a later successful evaluation. Explicit `?` propagation
  follows the language contract: return the same error from an enclosing value
  function, or propagate it to node failure in `when`. A returned Err alone
  cannot satisfy an escaping-error assertion.

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
| `6.0 /{zero: error} 2.0` | `ok(3.0)` |
| `1.0 /{zero: error} 0.0`; `0.0 /{zero: error} -0.0` | Err with code `numeric.division_by_zero` in both cases |
| NaN divided by a nonzero value with `zero: error` | Ok containing NaN |
| NaN divided by zero with `zero: error` | Err with code `numeric.division_by_zero` |
| Checked division publishes Err, then Ok on a later admitted evaluation | Two atomic result publications; no implicit node failure |
| `?` in a value function returning `result<f64, error>` receives checked-division Err | Same error returned; later statements are not executed |
| `?` inside `when` receives checked-division Err | Node failure with the same error; no successful result publication |
| `1.0 == 1.005`; `1.0 =={abs: 0.01} 1.005` | false; true |
| Relative-only tolerance comparing a nonzero value to zero, with rel less than one | false |
| NaN compared with itself by `<=>`, `==`, and `!=` | unordered; false; true |
| Opposite signed zeros compared by `<=>` | equivalent |
| Same signed infinities compared with a tolerance | true |
| `1e308 =={rel: 1.9} -1e308` | false, despite overflow in a naive difference/threshold calculation |
| Tolerance 1 applied to 0/0.75, 0.75/1.5, and 0/1.5 | true; true; false |

## Checked-division examples

Accepted value-function fragments, with the core `error` policy value in scope:

```hgl
const fn ratio(a: f64, b: f64) -> result<f64, error> =>
    a /{zero: error} b

const fn percentage(a: f64, b: f64) -> result<f64, error> {
    let quotient = (a /{zero: error} b)?
    return ok(quotient * 100.0)
}
```

`ratio(6.0, 2.0)` returns `ok(3.0)`. `ratio(1.0, 0.0)` returns Err with
`numeric.division_by_zero`. `percentage` propagates that same error or returns
the scaled quotient. Deliberate source rejection: a runtime-varying zero policy
cannot select between f64 and result return types. Executable conformance is
planned; these fragments declare no native entry points.
