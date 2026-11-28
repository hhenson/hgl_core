# Scalar operators and conversion

Status: OPS-12 is agreed; `hgl/` does not declare its default candidates
yet. Agreed operator domains for the scalar types other than numbers, the
rounding and range of a duration or an instant result, the comparison
outcome type, the domains of the conversion operator, what each conversion
yields, its options, and generic temporal identity handling. Runtime
conformance execution and the native ABI remain pending. The sentences of
OPS-2 and OPS-7 that begin "Proposed:" are proposed.

## Contract

The [numeric contract](numeric-operators.md) covers `i64` and `f64`. This
chapter covers the other scalar types. The
[language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/expressions-and-bindings.md)
owns symbol syntax, precedence, and phase checking. An instant is a
`datetime`, `date`, `civil_datetime`, or `zoned_datetime`.

- **OPS-1.** The symbol operators admit the operand types of the first table
  below, beside those of the numeric contract and of the
  [tenor contract](tenor-operations.md). Every other pairing is a
  source rejection, `operator.no_candidate`. An operation that commutes admits
  its operands in either order; one that does not admits the order shown. A
  division of a duration takes `zero` as NUM-12 says. A `str` and an `i64`
  give the text repeated that many times, the empty text for a count of
  zero; a count below zero is an HGL error with code
  `numeric.out_of_range`.
- **OPS-2.** `<`, `<=`, `>`, `>=`, and `<=>` compare two values of one type
  among `str`, `bytes`, `bool`, `date`, `time`, `datetime`,
  `civil_datetime`, and `duration`. Text is ordered by code point, so
  `"B" < "a"`, and `bytes` by unsigned octet (the language's TYPE-1), as
  text is. `false` is below `true`. An earlier `date`, `time`, or
  `datetime` is below a later one. A `duration` is ordered by its signed
  length, so `-5m < 0s`. Proposed: an earlier `civil_datetime` is below a
  later one. `==` and `!=` compare two values of one type. The zoned types
  have no order. Two `zoned_datetime` values are equal when their instants
  and their zone names both are; compared as UTC, they are equal when their
  instants are (the language's TYPE-2). The option is `utc`, a `bool`
  that is false unless given, `a =={utc: true} b`, declared by the
  candidates of `eq_` and `ne_` for two zoned datetimes. A `tenor`'s
  order is its own (TEN-3). Numbers
  compare as the numeric contract says, an `i64` with an `f64` included.
  Two structs that declare no `compare_` and no `eq_` of their own
  (OPS-12) are equal when they are of one struct with the same generic
  arguments (the language's TYPE-3), and each field is equal as `==` of the
  field's type has it. Two lists, and two tuples, are equal when they are
  of one length and each position is equal; two dictionaries when they have
  the same keys and each value is equal; two sets when they have the same
  members. NUM-5 holds inside a value: a struct, a list, a tuple, or a
  dictionary that holds a NaN is not equal to itself.
- **OPS-3.** `CmpResult` is the enum of comparison outcomes, with the members
  `less`, `equivalent`, `greater`, and `unordered`. `<=>` returns one
  (NUM-6). Of core's own types, only a numeric comparison and a tenor's
  (TEN-3) yield `unordered`; a candidate a program declares may. The other comparison forms return a bool.
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
  elements. The text of a number is all of the text: nothing stands before or
  after it, a space among them. The text a number converts to converts back
  to that number, a NaN to a NaN.
- **OPS-7.** A conversion is configured as division is (NUM-3, NUM-9).
  Options select how an `f64` is rounded to an `i64` and what a NaN becomes,
  an error or a sentinel value. A conversion can fail: an `f64` that is NaN,
  an infinity, or outside `i64` once rounded; text that is not what the
  third table reads as the type asked for; bytes that are not well-formed
  UTF-8. Proposed: an `i64` that is no member's number fails its
  conversion to an enum. Its error policy is
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
- **OPS-10.** A `duration` result is the exact result rounded to the nearest
  duration the type holds, a tie to the even one. The type's resolution is
  the language's (TYPE-2): a microsecond. A `duration` or an instant result
  outside its type's range is an HGL error with code `numeric.out_of_range`,
  as an `i64` result is (NUM-2): it neither wraps nor saturates. So is a
  `duration` multiplied or divided by a NaN or an infinity. A `date` with a
  `duration` added or taken away is the `civil_datetime` that far from the
  start of the date, whatever the duration. A `zoned_datetime` with a
  `duration` added or taken away is the instant that far away, in the same
  zone: its wall clock and offset are the zone's at that instant (the
  language's TYPE-2).
- **OPS-11.** `join` takes a `list<str>` in list order or a positional pack
  of strings in argument order (FN-8). It inserts `separator` between
  consecutive strings. The separator is a configuration parameter (FN-18)
  that defaults to one space. A connection given as the separator rejects
  (`connection.as_constant`, EXPR-14). With zero strings, the result is
  empty text. One string gives that string. A lifted call follows FN-6,
  ticking when an input ticks and every input is valid.
- **OPS-12.** `<=>` gives a type its order and its equality. Where
  `compare_` has a candidate for two types, core's default candidates of
  `eq_`, `lt_`, `le_`, `gt_`, and `ge_` for them read its outcome, as the
  numeric contract's table has it (NUM-6): `==` is true for `equivalent`,
  `<` for `less`, `<=` for `less` or `equivalent`, `>` for `greater`, and
  `>=` for `greater` or `equivalent`. `unordered` makes all five false.
  `ne_` is the complement of `eq_` wherever `eq_` has a candidate:
  `a != b` is `!(a == b)`. So a type declares `compare_` alone to have all
  seven, and `eq_` alone to have `==` and `!=` with no order. A more
  specific candidate of any of them is selected before the default (the
  language's GEN-5), and its own contract says what it gives. Core
  declares three kinds of `eq_` more specific than the default: for
  numbers, which with no tolerance given is the outcome of `<=>` and with
  one is NUM-4's closeness; for two zoned datetimes, which have no
  `compare_` and take `utc` (OPS-2); and for two values of one type that
  has no candidate of `compare_`, OPS-2's `==`. Core's `ne_` carries the
  options core's `eq_` declares, `abs`, `rel`, and `utc`, and no other: a
  candidate of `eq_` that declares an option of its own has a configured
  `!=` only where a matching candidate of `ne_` is declared. The defaults
  are HGL. A provider supplies `compare_` for core's ordered types and
  those three kinds of `eq_`, and no other comparison.

  Accepted illustration, a fragment; conformance planned:

  ```hgl
  struct Grade { label: str, rank: i64? }

  impl const fn compare_(lhs: Grade, rhs: Grade) -> CmpResult {
      if lhs.rank == null || rhs.rank == null { return .unordered }
      return lhs.rank <=> rhs.rank
  }

  test declared_order {
      let pass = Grade(label: "pass", rank: 1)
      let merit = Grade(label: "merit", rank: 1)
      let absent = Grade(label: "absent", rank: null)
      assert pass == merit && !(pass != merit)     # the outcome, not the fields
      assert pass <= merit && !(pass < merit)
      assert !(absent == absent) && absent != absent
      assert !(absent < pass) && !(absent >= pass)
  }
  ```

| Operator | Operands | Result |
| --- | --- | --- |
| `+` | `str` and `str` | `str` |
| `+` | `duration` and `duration` | `duration` |
| `+` | A `datetime`, `civil_datetime`, or `zoned_datetime` and a `duration`, in either order | That instant's type |
| `+` | `date` and `duration`, in either order | `civil_datetime` |
| `+` | `date` and `time`, in either order | `civil_datetime` |
| `-` | `duration` less `duration` | `duration` |
| `-` | A `datetime`, `civil_datetime`, or `zoned_datetime` less a `duration` | That instant's type |
| `-` | `date` less a `duration` | `civil_datetime` |
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
| `i64` | `f64`, `bool`, `str`, an enum |
| `f64` | `i64`, `bool`, `str` |
| `bool` | `i64`, `f64`, `str` |
| `str` | `i64`, `f64`, `bool`, `bytes` |
| `bytes` | `str` |
| `date` | `datetime` |
| `datetime` | `date` |
| `tenor` | `str` |
| `Type` | `str` |
| An enum | `i64`, `str` |
| A list, a set, a dictionary, a tuple, a struct | `str` |
| A list, set or dictionary delta | `str` |

| Conversion | Yields |
| --- | --- |
| `i64` to `f64` | The nearest `f64` |
| `f64` to `i64` | The integer its rounding selects (OPS-7) |
| `i64` or `f64` to `bool` | False for zero, true otherwise; NaN is true |
| `bool` to `i64` or `f64` | 0 or 1 |
| `i64` to `str` | Its decimal digits, after a `-` where it is below zero: `42`, `-3` |
| `f64` to `str` | The shortest decimal text that reads back as the same `f64`, always with a `.` or an exponent, after a `-` where its sign is negative: `3.0`, `0.1`, `-2.5`, `-0.0`. It has an exponent where the number is 1e16 or more, or below 1e-4 and not zero, in size: `1e21`, `1.5e-7`. A NaN is `nan`, and the infinities are `inf` and `-inf` |
| `bool` to `str` | `true` or `false` |
| `str` to `i64` | The number the text writes: an integer literal, after one optional `-`: `"42"`, `"-3"`, `"0x1F"`, `"1_000"`. A number outside `i64` fails |
| `str` to `f64` | The `f64` nearest the number the text writes, a tie to the even one: a float or an integer literal, after one optional `-`: `"2.5"`, `"-1e5"`, `"3"`. One that rounds to an infinity fails, as its literal is rejected (the language's LEX-20). `"nan"` is a NaN, and `"inf"` and `"-inf"` are the infinities |
| `str` to `bool` | True for `"true"` and false for `"false"` |
| `str` to `bytes`, `bytes` to `str` | UTF-8 |
| `date` to `datetime` | The start of that date, in UTC |
| `datetime` to `date` | The date of the instant, in UTC |
| `tenor` to `str` | Its text, in its largest units (TEN-9) |
| `Type` to `str` | Its canonical diagnostic spelling (TYPE-18, FMT-5) |
| An enum to `str` | The member's name: `"buy"` for `Side.buy` (the language's TYPE-7) |
| An enum to `i64` | The member's number |
| `i64` to an enum | The member of that number |
| A list, a set, a dictionary, a tuple or a struct to `str` | The value as a program writes it, a `str` among its parts in quotes: `[1, 2]`, `["a": 1]`, `(1, "a")`, `Point(x: 1.0, y: 2.0)` ([formatting](formatting.md), FMT-5) |
| A list, set or dictionary delta to `str` | Its TYPE-16 constructor, omitting empty or null properties (FMT-5) |

| Option | Applies to | Members | Default |
| --- | --- | --- | --- |
| `rounding` | `f64` to `i64` | `.truncate`, toward zero; `.floor`, toward negative infinity; `.ceiling`, toward positive infinity; `.half_even`, to the nearest, a tie to the even integer; `.half_up`, to the nearest, a tie away from zero; `.half_down`, to the nearest, a tie toward zero | `.truncate` |
| `nan` | `f64` to `i64` | An `i64?`: the value a NaN converts to. Null leaves a NaN to `invalid` | null |
| `invalid` | Every conversion that can fail (OPS-7) | `.error`: the conversion returns `result<T, error>`, and a failure is an Err carrying error code `convert.invalid`. `.fail`: it returns T, and a failure is an escaping HGL error with that code | `.error` |

| Valid inputs under `invalid: .identity` (OPS-9) | Addition | Subtraction |
| --- | --- | --- |
| Both | `add_(lhs, rhs)` | `sub_(lhs, rhs)` |
| Left only | `lhs` | `lhs` |
| Right only | `rhs` | `neg_(rhs)` |
| Neither | Invalidate the output | Invalidate the output |

## Examples

The first call joins a list with a supplied separator. The second joins
positional strings with the default separator (OPS-11).

```hgl
join{separator: ", "}(["a", "b"])
join("a", "b", "c")
```

They give `"a, b"` and `"a b c"` respectively.

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

A duration result rounds to the type's resolution, and one out of range is
an error (OPS-10):

| Expression | Result |
| --- | --- |
| `5m / 3` | `100s`, exactly |
| `1us * 0.5`; `3us * 0.5` | `0us`; `2us`: half a microsecond is a tie, and goes to the even one |
| `1us / 3`; `2us / 3` | `0us`; `1us` |
| A duration times an infinity or a NaN | HGL error with code `numeric.out_of_range` |
| `@9999-12-31T23:59:59Z + 1d` | HGL error with code `numeric.out_of_range` |

Text is repeated a count, in either order (OPS-1):

| Expression | Result |
| --- | --- |
| `"ab" * 2`; `2 * "ab"` | `"abab"` |
| `"ab" * 0` | `""` |
| `"ab" * -1` | HGL error with code `numeric.out_of_range` |

A date and a duration give a civil datetime (OPS-1, OPS-10):

| Expression | Result |
| --- | --- |
| `@2026-09-03 + 2d`; `2d + @2026-09-03` | `@2026-09-05T00:00` |
| `@2026-09-03 + 36h` | `@2026-09-04T12:00` |
| `@2026-09-03 - 1h` | `@2026-09-02T23:00` |

A duration moves a zoned datetime's instant (OPS-10), and two zoned
datetimes are equal by instant and zone name (OPS-2). London's clocks go
forward an hour at 01:00 UTC on 29 March 2026:

| Expression | Result |
| --- | --- |
| `@2026-03-28T12:00+00:00[Europe/London] + 1d` | `@2026-03-29T13:00+01:00[Europe/London]`: 86,400 seconds later |
| `@2026-03-29T00:30+00:00[Europe/London] + 1h` | `@2026-03-29T02:30+01:00[Europe/London]` |
| `@2026-09-03T09:30+01:00[Europe/London] == @2026-09-03T08:30+00:00[UTC]` | false: one instant, two zones |
| `@2026-09-03T09:30+01:00[Europe/London] =={utc: true} @2026-09-03T08:30+00:00[UTC]` | true |

The [zoned datetime cases](https://github.com/hhenson/hgl_core/blob/main/tests/core_conformance/zoned_datetimes.hgl)
check these.

Accepted illustration of equality between values that hold others (OPS-2),
complete module:

```hgl
module examples.equality

struct Point {
    x: f64
    y: f64
}

abstract struct Shape {}

struct Tagged<T> : Shape {
    tag: T
}

const fn same(lhs: Shape, rhs: Shape) -> bool => lhs == rhs

test equality {
    assert Point(x: 1.0, y: 2.0) == Point(x: 1.0, y: 2.0)
    let odd = Point(x: 0.0 / 0.0, y: 0.0)
    assert odd != odd
    assert [odd.x] != [odd.x]
    assert !same(Tagged(tag: 1), Tagged(tag: 1.0))
}
```

`odd` holds a NaN, so it is not equal to itself, and neither is a list of
that NaN. A `Tagged<i64>` is no `Tagged<f64>`, so the two are not equal where
they meet as values of their parent. Executable conformance is planned.

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
