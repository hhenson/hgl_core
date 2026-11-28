# Numeric operators and policies

Status: agreed baseline, explicit operator configuration, approximate equality,
full comparison, IEEE division default, the zero policy of every division,
and checked-division result contract. No executable conformance or native ABI
is specified.

## Contract

The [language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/expressions-and-bindings.md)
owns symbol syntax, configuration binding, precedence, and phase checking.
Core owns numerical domains, public configuration, results, and errors.

- **NUM-1.** For i64 operands, `+`, `-`, `*`, `//`, and `%` return i64.
  Unconfigured mixed i64/f64 and f64 arithmetic returns f64; `/` defaults to f64.
  NUM-9 defines the configured division result types.
  Unary negation preserves the numeric type. Integer floor division rounds
  toward negative infinity; modulo has the divisor's sign when nonzero. So
  do `//` and `%` of finite f64 operands. Where an f64 operand is an
  infinity or NaN and the divisor is not zero, `//` is the floor of the
  IEEE quotient and `%` what IEEE `fmod` gives: a remainder by an infinity
  is the numerator as it is, whatever its sign.
- **NUM-2.** An unrepresentable i64 result produces an HGL error with code
  `numeric.out_of_range`; it neither wraps nor silently promotes. Floating
  arithmetic rounds to binary64 and
  preserves IEEE special values, including signed zero and overflow infinity.
  Mixed numeric comparison must not first round an i64 operand to f64 and
  thereby lose a distinction between the operand values.
- **NUM-3.** The `zero` option of a division is a `ZeroPolicy`, an enum with
  members `ieee`, `nan`, `error`, and `fail`, written `.ieee`, `.nan`,
  `.error`, and `.fail` where the option is given. True division `/`, whose
  numeric result is f64, defaults to the `ieee` zero policy:
  a finite nonzero numerator divided by signed zero produces signed infinity;
  zero divided by zero produces NaN. The infinity sign combines both operand
  signs. The `nan` zero policy instead produces NaN for every zero divisor.
  The `error` alternative follows NUM-9. The `fail` policy makes a zero
  divisor an HGL error with code `numeric.division_by_zero`. The zero
  option applies to zero divisors, not to unrelated operand errors. Policies
  requiring NaN/infinity need a result type that can represent those values
  (NUM-12).
- **NUM-4.** Plain numeric `==` is exact and `!=` is its complement. Configured
  `==` accepts absolute (`abs`) and relative (`rel`) tolerance, each defaulting
  to zero. For finite operands, closeness is the symmetric condition below.
  Tolerances must be finite and nonnegative. Configured `!=` is the complement
  under the same options. An invalid tolerance, negative or not finite, is
  an HGL error with code `numeric.invalid_tolerance`, under the ordinary
  phase rules.

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
  `<=>`, returns an ordinary comparison outcome, a
  [`CmpResult`](scalar-operators.md): `less`, `equivalent`,
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
  contract. An option is declared by the candidates that use it, and an
  omitted option takes the default they declare; imports, target choice,
  and optional folding must not silently alter it. Value and temporal
  execution preserve the same numerical outcomes under their existing phase,
  lifting, validity, publication, and error rules. A NaN/infinity publication
  is present data. Required-constant errors and optional folding follow the
  language's error-phase contract.
- **NUM-9.** Division with `zero: .ieee` or `zero: .nan` returns f64; division
  with `zero: .error` returns `result<f64, error>`. A division whose result
  is another type T (NUM-12) returns `result<T, error>` under `zero: .error`.
  Resolve the zero policy during source checking because it determines the
  result type: `zero` is a generic value parameter of the candidates that
  declare it, and a call gives it a compile-time constant. For the error
  policy,
  either signed zero divisor produces `err(...)` carrying error code
  `numeric.division_by_zero`; every nonzero divisor produces `ok(value)` from
  the ordinary division. This includes successful NaN/infinity results where
  IEEE division produces them. This policy checks only the divisor and does
  not catch unrelated errors evaluating the operands.
- **NUM-10.** A checked-division Err is ordinary result data. It neither
  implicitly fails a node nor requires an exception. Temporal results publish
  atomically under the existing admission rules; an Err is a present publication
  and does not prevent a later successful evaluation. Explicit `?` propagation
  follows the language contract: return the same error from an enclosing value
  function, or propagate it to node failure in `when`. A returned Err alone
  cannot satisfy an escaping-error assertion.
- **NUM-11.** `abs` and `rel` are configuration parameters of the numeric
  candidates of `eq_` and `ne_` (the language's FN-18), each an `f64`. They
  are written in the braces of either spelling, `a =={abs: 0.01} b` and
  `eq_{abs: 0.01}(a, b)`, and a named argument in the parentheses does not
  name one. In temporal composition they bind during graph construction,
  and a connection given to one rejects (`connection.as_constant`); in a
  value function or runtime hook they are values for the current
  invocation. They create no temporal dependency (the language's EXPR-14).
  Configuration order and phase checking follow EXPR-12 through EXPR-14 in
  the language specification.
- **NUM-12.** Every division takes `zero`: `/`, `//`, and `%`, of numbers and
  of durations (OPS-1). The policies one admits, and its default, follow
  from what its result can hold, as the second table below says. A policy a
  division does not admit is a source rejection, `operator.no_candidate`.
  Under `.ieee` a floor division or a modulo with an `f64` operand is what
  IEEE arithmetic makes of the zero divisor.

| Numeric expression | Full-comparison outcomes producing true |
| --- | --- |
| `a == b` | `equivalent` |
| `a != b` | `less`, `greater`, `unordered` |
| `a < b` | `less` |
| `a <= b` | `less`, `equivalent` |
| `a > b` | `greater` |
| `a >= b` | `greater`, `equivalent` |

| A division whose result is | Namely | Admits | Default |
| --- | --- | --- | --- |
| An `f64` | `/` of numbers; `//` and `%` with an `f64` operand; a `duration` by a `duration` | `.ieee`, `.nan`, `.error`, `.fail` | `.ieee` |
| An `i64` or a `duration` | `//` and `%` of two `i64`; a `duration` by a number; `//` and `%` of two durations (OPS-13) | `.error`, `.fail` | `.fail` |

## Examples and conformance observations

Accepted expression fragments; `.ieee` and `.nan` name members of the `zero`
option's enum, not new numeric literals:

```hgl
1.0 / 0.0
1.0 /{zero: .ieee} 0.0
1.0 /{zero: .nan} 0.0
1.0 =={abs: 0.01} 1.005
1.0 <=> 2.0
```

Core owns the following expected observations. Executable cases are planned.
NaN/infinity names in the table describe outcomes, not source literal syntax.

| Case | Expected observation |
| --- | --- |
| `7 / 2`; `-7 // 3`; `-7 % 3` | `3.5`; `-3`; `2` |
| Maximum i64 plus one; negation of minimum i64 | HGL errors with code `numeric.out_of_range`, without wrapping |
| `9007199254740993 == 9007199254740992.0` | false |
| `1.0 / 0.0`; `1.0 / -0.0`; `0.0 / 0.0` | Positive infinity; negative infinity; NaN |
| `1.0 /{zero: .nan} 0.0` | NaN |
| `6.0 /{zero: .error} 2.0` | `ok(3.0)` |
| `1.0 /{zero: .error} 0.0`; `0.0 /{zero: .error} -0.0` | Err with code `numeric.division_by_zero` in both cases |
| NaN divided by a nonzero value with `zero: .error` | Ok containing NaN |
| NaN divided by zero with `zero: .error` | Err with code `numeric.division_by_zero` |
| `1.0 /{zero: .fail} 0.0` | HGL error with code `numeric.division_by_zero` |
| `7 // 0`; `7 % 0`; `5m / 0`; `5m // 0s`; `5m % 0s` | HGL error with code `numeric.division_by_zero` in each case |
| `7m % 3m`; `-7m % 3m`; `7m % -3m`; `5m %{zero: .error} 0s` | `1m`; `2m`; `-2m`; Err with code `numeric.division_by_zero` |
| `7 //{zero: .error} 2`; `7 //{zero: .error} 0` | `ok(3)`; Err with code `numeric.division_by_zero` |
| `7 //{zero: .nan} 2`; `5m /{zero: .ieee} 2` | Source rejection in both cases: the result holds no NaN or infinity |
| `7.0 // 0.0`; `7.0 % 0.0`; `5m / 0s` | Positive infinity; NaN; positive infinity |
| Positive infinity `// 2.0`; positive infinity `% 2.0` | Positive infinity; NaN |
| `1.0 //` positive infinity; `-1.0 //` positive infinity | `0.0`; `-0.0` |
| `1.0 %` positive infinity; `-1.0 %` positive infinity | `1.0`; `-1.0` |
| Infinity `//` infinity; `//` or `%` with a NaN operand | NaN; NaN |
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
| `left() =={abs: tolerance()} right()` with value helpers recording their names | Invocation order left, tolerance, right; each once |
| `eq_{abs: tolerance()}(left(), right())` with the same value helpers | Invocation order tolerance, left, right; each once |
| Scalar tolerance helper configuring a temporal comparison over several ticks | Helper called once during graph construction; bound tolerance reused |
| Tolerance helper inside a runtime-hook comparison evaluated several times | Helper called for each reached comparison evaluation |
| Temporal endpoint supplied to the scalar `abs` option in composition, in either spelling | Source rejection, `connection.as_constant`; no implicit extra temporal input |

## Checked-division examples

Accepted value-function fragments:

```hgl
const fn ratio(a: f64, b: f64) -> result<f64, error> =>
    a /{zero: .error} b

const fn percentage(a: f64, b: f64) -> result<f64, error> {
    let quotient: f64 = a /{zero: .error} b
    return ok(quotient * 100.0)
}
```

`ratio(6.0, 2.0)` returns `ok(3.0)`. `ratio(1.0, 0.0)` returns Err with
`numeric.division_by_zero`. `percentage` propagates that same error or returns
the scaled quotient. Deliberate source rejection: a runtime-varying zero policy
cannot select between f64 and result return types. Executable conformance is
planned; these fragments declare no native entry points.
