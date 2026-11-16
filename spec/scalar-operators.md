# Scalar operators and conversion

Status: agreed operator domains for the scalar types other than numbers, the
comparison outcome type, the domains of the conversion operator, what each
conversion yields, its options, and generic temporal identity handling.
Runtime conformance execution and the native ABI remain pending.

## Contract

The [numeric contract](numeric-operators.md) covers `i64` and `f64`. This
chapter covers the other scalar types. The
[language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/expressions-and-bindings.md)
owns symbol syntax, precedence, and phase checking. An instant is a
`datetime`, `date`, `civil_datetime`, or `zoned_datetime`.

- **OPS-1.** The symbol operators admit the operand types of the first table
  below, beside those of the numeric contract. Every other pairing is a
  source rejection, `operator.no_candidate`. An operation that commutes admits
  its operands in either order; one that does not admits the order shown. A
  division of a duration takes `zero` as NUM-12 says.
- **OPS-2.** `<`, `<=`, `>`, `>=`, and `<=>` compare two values of one type
  among `str`, `bool`, `date`, `time`, `datetime`, `civil_datetime`, and
  `duration`. `==` and `!=` compare two values of one type. The zoned types
  have no order: same-instant zoned values need not be equal. Numbers
  compare as the numeric contract says, an `i64` with an `f64` included.
- **OPS-3.** `CmpResult` is the enum of comparison outcomes, with the members
  `less`, `equivalent`, `greater`, and `unordered`. `<=>` returns one
  (NUM-6). Only a numeric comparison yields `unordered`. The other
  comparison forms return a bool.
- **OPS-4.** `convert<T>(value)` converts a value to the type T. An integer
  literal is an `f64` where one is expected; no other value changes type
  without it. It admits the conversions of the second table below and
  rejects every other.
- **OPS-5.** `&&`, `||`, and `!` take bools. A number or a string has no
  truth value. A chain of three or more operands joined by `&&` alone is
  `all_` over them, and one joined by `||` alone is `any_`. Core's candidate
  of each for bools is the chain itself, `and_` or `or_` applied left to
  right, short-circuiting as the links do.
- **OPS-6.** A conversion yields what the third table below says.
  `convert<list<B>>(values)` is admitted too, where `values` is a `list<A>`
  and the struct `A` inherits from the abstract struct `B`; it yields the same
  elements.
- **OPS-7.** A conversion is configured as division is (NUM-3, NUM-9).
  Options select how an `f64` is rounded to an `i64` and what a NaN becomes,
  an error or a sentinel value. A conversion can fail: an `f64` that is NaN,
  an infinity, or outside `i64` once rounded; text that is no literal of the
  type asked for; bytes that are not well-formed UTF-8. Its error policy is
  checked wherever it can be: the conversion then returns `result<T, error>`,
  and a failure is an Err, ordinary result data (NUM-10). A policy under
  which a failure escapes instead is an option.
- **OPS-8.** The options of a conversion are those of the fourth table below.
  Each is given as a named argument, and declared by the conversions that
  use it. `rounding` is a `Rounding` and `invalid` an `InvalidPolicy`, each
  an enum of the members shown. `invalid` determines the result type and is
  resolved during source checking, as NUM-9 does for division: it is a
  generic value parameter of those conversions, and a call gives it a
  compile-time constant. `rounding` and `nan` are parameters. A conversion
  that cannot fail returns T.
- **OPS-9.** Same-type temporal addition and subtraction may select an
  `invalid: .identity`, a const generic `InvalidInputPolicy` whose member is
  `identity`. Addition requires a value `add_(T, T) -> T`;
  subtraction requires value `sub_(T, T) -> T` and `neg_(T) -> T`.
  The option admits no mixed operand or result types. It adds no scalar
  native entry points. Omission keeps ordinary candidate selection and
  lifting and validity admission. These temporal candidates require the
  explicit `.identity` option.
  Under `.identity`, use the table below. Input invalidation activates the
  node as END-4 specifies; silence retains input values and creates no work.
  Valid zero, empty text, null, and NaN remain values, not invalid inputs.

| Operator | Operands | Result |
| --- | --- | --- |
| `+` | `str` and `str` | `str` |
| `+` | `duration` and `duration` | `duration` |
| `+` | An instant and a `duration`, in either order | The instant's type |
| `+` | `date` and `time` | `civil_datetime` |
| `-` | `duration` less `duration` | `duration` |
| `-` | An instant less a `duration` | The instant's type |
| `-` | `datetime` less `datetime`, `date` less `date`, `civil_datetime` less `civil_datetime` | `duration` |
| `*` | `duration` and `i64` or `f64`, in either order | `duration` |
| `*` | `str` and `i64`, in either order | `str` |
| `/` | `duration` by `i64` or `f64` | `duration` |
| `/` | `duration` by `duration` | `f64` |
| `//` | `duration` by `duration` | `i64` |
| `-x` | `duration` | `duration` |
| `<`, `<=`, `>`, `>=` | Two of one ordered type (OPS-2) | `bool` |
| `<=>` | Two of one ordered type (OPS-2) | `CmpResult` |
| `==`, `!=` | Two values of one type | `bool` |

| `convert` from | To |
| --- | --- |
| `i64` | `f64`, `bool`, `str` |
| `f64` | `i64`, `bool`, `str` |
| `bool` | `i64`, `f64`, `str` |
| `str` | `i64`, `f64`, `bool`, `bytes` |
| `bytes` | `str` |
| `date` | `datetime` |
| `datetime` | `date` |

| Conversion | Yields |
| --- | --- |
| `i64` to `f64` | The nearest `f64` |
| `f64` to `i64` | The integer its rounding selects (OPS-7) |
| `i64` or `f64` to `bool` | False for zero, true otherwise; NaN is true |
| `bool` to `i64` or `f64` | 0 or 1 |
| A number or a `bool` to `str` | The text of the literal that denotes the value |
| `str` to `i64`, `f64`, or `bool` | The value of the literal the text is |
| `str` to `bytes`, `bytes` to `str` | UTF-8 |
| `date` to `datetime` | The start of that date, in UTC |
| `datetime` to `date` | The date of the instant, in UTC |

| Option | Applies to | Members | Default |
| --- | --- | --- | --- |
| `rounding` | `f64` to `i64` | `.truncate`, toward zero; `.floor`, toward negative infinity; `.ceiling`, toward positive infinity; `.half_even`, to the nearest, a tie to the even integer; `.half_up`, to the nearest, a tie away from zero; `.half_down`, to the nearest, a tie toward zero | `.truncate` |
| `nan` | `f64` to `i64` | An `i64?`: the value a NaN converts to. Null leaves a NaN to `invalid` | null |
| `invalid` | Every conversion that can fail (OPS-7) | `.error`: the conversion returns `result<T, error>`, and a failure is an Err carrying error code `convert.invalid`. `.fail`: it returns T, and a failure is an escaping HGL error | `.error` |

| Valid inputs under `invalid: .identity` (OPS-9) | Addition | Subtraction |
| --- | --- | --- |
| Both | `add_(lhs, rhs)` | `sub_(lhs, rhs)` |
| Left only | `lhs` | `lhs` |
| Right only | `rhs` | `neg_(rhs)` |
| Neither | Invalidate the output | Invalidate the output |

## Examples

The [wired identity example](temporal-arithmetic.md) shows OPS-9 through
publication, invalidation, silence, and recovery.

Accepted expression fragments, with `count` an `i64` in scope:

```hgl
"a" + "b"
5m + 30s
@2026-09-03T09:30Z + 5m
5m + @2026-09-03T09:30Z
@2026-09-04 - @2026-09-03
2 * 5m
5m / 2m
5m <=> 30s
convert<f64>(count)
```

Their types are `str`, `duration`, `datetime`, `datetime`, `duration`,
`duration`, `f64`, `CmpResult`, and `f64`. Deliberate source rejections under
OPS-1: `5m - @2026-09-03T09:30Z`, a duration less an instant; `"a" + 1`;
`@2026-09-03 + @2026-09-04`. Under OPS-5: `1 && true`. Executable conformance
is planned.

Accepted expression fragments for OPS-8, with `price` an `f64` in scope:

```hgl
convert<i64>(price)
convert<i64>(price, rounding: .half_even)
convert<i64>(price, nan: 0)
convert<i64>(price, invalid: .fail)
convert<f64>("1.5")
```

Their types are `result<i64, error>` for the first three, `i64`, and
`result<f64, error>`. With `nan: 0` a NaN converts to 0; an infinity still
fails. Deliberate source rejection: `convert<i64>(price, invalid: policy)`
with `policy` no constant, which leaves the result type unresolved.
