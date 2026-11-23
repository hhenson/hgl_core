# Tenors and calendars

Status: TEN-1 through TEN-9 are agreed.

## Scope and API

The [language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/types-and-values.md)
says what a `tenor` is and how one is written (TYPE-17, LEX-28): a count of
months and a count of days, or a count of business days. This contract has
its operators, how each form moves a date, what a business day is, and how
a moved date is rolled onto one.

| Core API | Contract |
| --- | --- |
| `business_day<C>(calendar: C, day: date) -> bool` | An operator. A calendar is any value it has a candidate for |
| `roll<R, C>(rolling_rule: R, calendar: C, day: date) -> date` | An operator. A rolling rule is any value it has a candidate for |
| `Weekdays` | A struct with no field: the calendar on which every day but a Saturday and a Sunday is a business day |
| `Rolling` | An enum: `following`, `modified_following`, `preceding`, `modified_preceding` |

The operators and the types are declared in `hgl/core/calendar.hgl`, with
the candidates that move a date. A tenor's own arithmetic, order and text
are native, in `hgl/native/tenors.hgl`.

## Rules

- **TEN-1.** The symbol operators admit the operand types of the first
  table below. Every other pairing is a source rejection,
  `operator.no_candidate`. Scaling commutes, and admits its operands in
  either order; a date is the left operand.
- **TEN-2.** A sum, a difference, a negation, and a scaling apply to each
  count. The operands of a sum or a difference are of one form: a tenor of
  business days with one of months and days is an HGL error with code
  `tenor.mixed_units`. A count outside the i64 range is an HGL error with
  code `numeric.out_of_range`.
- **TEN-3.** The order is count by count. One tenor is less than or equal
  to another when they are of one form and no count of it is greater. `<=>`
  gives `equivalent` for two equal tenors; `less` or `greater` where each
  count that differs differs that way; and `unordered` otherwise, two
  tenors of different forms included. `<`, `<=`, `>`, and `>=` are false for
  an unordered pair. Where `a <= b`, `day + a <= day + b` for every date
  and calendar.
- **TEN-4.** A date moved by a tenor of months and days takes the months
  first: the same day of the month that many months on, or the last day of
  that month where it has no such day. Then it takes the days. A count
  below zero moves back. `day - tenor` is `day + (-tenor)`. A result outside
  the `date` range is an HGL error with code `numeric.out_of_range`. Only
  the result is held to the range: the date the months give may lie outside
  it where the days bring the result back within it.
- **TEN-5.** A calendar says which dates are business days. It is any value
  for which a value `business_day(calendar, date) -> bool` exists. `+` and
  `-` of a date and a tenor take one as the option `calendar`. Where none is
  given the calendar is `Weekdays()`. `calendar` is a parameter of those
  candidates, so a graph may give it a connection, and the date is then
  worked out again when the calendar changes, in either spelling:
  `day +{calendar: open_days} 2B` and `add_(day, 2B, calendar: open_days)`
  (the language's EXPR-14, FN-6).
- **TEN-6.** A count of business days starts from a business day. For a
  count n that is not negative, `day + tenor` is the business day on or
  after `day`, and then the nth business day after that; `day - tenor` is
  the business day on or before `day`, and then the nth before that. So
  `day + 0B` is the business day on or after `day`, `day - 0B` the one on or
  before, and every result is a business day. A count below zero moves the
  other way: `day + (-tenor)` is `day - tenor`.
- **TEN-7.** A rolling rule says which business day a date becomes. It is
  any value for which a value `roll(rolling_rule, calendar, date) -> date`
  exists. `+` and `-` of a date and a tenor take one as the option
  `rolling_rule`: the result is `roll(rolling_rule, calendar, moved)`, where
  `moved` is the date the tenor gives (TEN-4, TEN-6) and `calendar` the one
  the operation has (TEN-5). With no rolling rule the result is `moved`.
- **TEN-8.** `Rolling` has the rules of the second table below, for any
  calendar.
- **TEN-9.** `convert<str>(tenor)` gives a tenor's text, which uses the
  largest units: its months as `Y` and `M`, its days as `W` and `D`, or its
  business days as `B`. A unit whose count is zero is not written; a tenor
  of no months and no days is `0D`. A count below zero has `-` before its
  units.

| Operator | Operands | Result |
| --- | --- | --- |
| `+`, `-` | `date` and `tenor`, the date on the left | `date` |
| `+`, `-` | `tenor` and `tenor` | `tenor` |
| `*` | `tenor` and `i64`, in either order | `tenor` |
| `-x` | `tenor` | `tenor` |
| `<`, `<=`, `>`, `>=` | `tenor` and `tenor` | `bool` |
| `<=>` | `tenor` and `tenor` | `CmpResult` |
| `==`, `!=` | `tenor` and `tenor` | `bool` |

| `Rolling` | The business day a date becomes |
| --- | --- |
| `following` | The one on or after it |
| `preceding` | The one on or before it |
| `modified_following` | The one on or after it, or where that is in another month the one on or before |
| `modified_preceding` | The one on or before it, or where that is in another month the one on or after |

## Examples

Accepted expression fragments, with `lme` a calendar and `lme_rolling` a
rolling rule in scope:

```hgl
@2026-01-31 + 3M
@2026-01-02 + 2B
@2026-01-02 +{calendar: lme} 2B
@2026-01-31 +{rolling_rule: Rolling.modified_following} 3M
@2026-01-31 +{rolling_rule: lme_rolling, calendar: lme} 3M
6M + 6M
2 * 1Y6M
1M <=> 45D
convert<str>(13D)
```

Their types are `date` for the first five, `tenor`, `tenor`, `CmpResult`,
and `str`. Deliberate source rejections under TEN-1: `3M + @2026-01-31`, a
tenor on the left of a date; `@2026-01-31T09:30Z + 3M`, an instant that is
no date; `3M / 2`.

A date and a tenor of months and days (TEN-4):

| Expression | Result |
| --- | --- |
| `@2026-01-31 + 1M` | `@2026-02-28`: February has no 31st |
| `@2026-01-31 + 1Y6M` | `@2027-07-31` |
| `@2026-01-31 + 1M1D` | `@2026-03-01`: the month first, then the day |
| `@2026-01-31 + 2 * 1M` | `@2026-03-31` |
| `@2026-01-31 + 1M + 1M` | `@2026-03-28`: two moves, each from where the last one ended |
| `@2026-01-31 - 2W` | `@2026-01-17` |
| `@9999-12-15 + (1M - 30D)` | `@9999-12-16`: the months alone pass the last date, and the days bring the result back |
| `@9999-12-15 + 1M` | HGL error with code `numeric.out_of_range` |

A date and a count of business days, with no calendar (TEN-5, TEN-6).
2026-01-02 is a Friday and 2026-01-03 a Saturday:

| Expression | Result |
| --- | --- |
| `@2026-01-02 + 1B` | `@2026-01-05`, the Monday |
| `@2026-01-03 + 0B` | `@2026-01-05`: on to a business day |
| `@2026-01-03 - 0B` | `@2026-01-02`: back to one |
| `@2026-01-03 + 1B` | `@2026-01-06`: on to the Monday, then one more |
| `@2026-01-03 + 2B` | `@2026-01-07`: a trade at the weekend, settled in two days |

A tenor's own arithmetic, order, and text (TEN-2, TEN-3, TEN-9):

| Expression | Result |
| --- | --- |
| `6M + 6M == 1Y`; `2 * 1Y6M == 3Y` | True |
| `2B + 1M` | HGL error with code `tenor.mixed_units` |
| `1M < 2M`; `1M < 1M1D` | True |
| `1M < 45D`; `45D < 1M` | False: the two have no order |
| `1M <=> 45D`; `2B <=> 1M` | `unordered` |
| `convert<str>(13D)`; `convert<str>(18M)` | `"1W6D"`; `"1Y6M"` |
| `convert<str>(-(13D))`; `convert<str>(1M - 1D)` | `"-1W6D"`; `"1M-1D"` |

Accepted illustration, complete module: a calendar and a rolling rule of a
library's own.

```hgl
module examples.calendars

struct Holidays {
    first: date
    second: date
}

impl const fn business_day(calendar: Holidays, day: date) -> bool =>
    day != calendar.first && day != calendar.second && business_day(Weekdays(), day)

struct Unrolled {}

impl const fn roll<C>(rolling_rule: Unrolled, calendar: C, day: date) -> date => day

const fn settles(trade: date, calendar: Holidays) -> date =>
    trade +{calendar: calendar} 2B

const fn matures(spot: date, calendar: Holidays) -> date =>
    spot +{rolling_rule: Rolling.modified_following, calendar: calendar} 3M

test calendars {
    let closed = Holidays(first: @2026-01-05, second: @2026-04-30)
    assert settles(@2026-01-02, closed) == @2026-01-07
    assert matures(@2026-01-30, closed) == @2026-04-29
    assert @2026-01-31 +{rolling_rule: Unrolled()} 0D == @2026-01-31
}
```

`Holidays` adds two closed days to the weekend, by asking `Weekdays()` for
the rest. `settles` counts two business days on it: from Friday the 2nd,
past the weekend and the closed Monday, to Wednesday the 7th. `matures`
moves three months to the 30th of April, which is closed, and rolls: the
business day after it is in May, so the rule gives the one before. A graph
gives `calendar` a connection, and the date is worked out again when the
calendar changes (the language's FN-3).
