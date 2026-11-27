# Formatting

Status: FMT-1 through FMT-9 are agreed. The [reference](#reference)
lists every form in one place. The operator, its value candidate and `Marked` are declared in
[`hgl/native/formatting.hgl`](../hgl/native/formatting.hgl).
The scalar graph candidate is in [hgl/core/formatting.hgl](../hgl/core/formatting.hgl).

## Scope and API

A format string is text with places for values. One rule serves every
function of core that takes one: `format`, which gives the text, and
`debug_print` ([diagnostics](diagnostics.md)), which writes it. The language
specification says what a pack is and how a call gives one
([functions](https://github.com/hhenson/hgl_spec/blob/main/spec/language/functions-and-phases.md),
FN-8).

| Core API | Contract |
| --- | --- |
| `format{options}(fmt_str, args..., kwargs...)` | The text of `fmt_str` with its places filled |
| `Marked<T>` | A value of T or none, and whether it ticked, for a place to show |

The native declaration gives the value candidate and `Marked<T>`'s fields.
The graph candidate's signature is shown here for reference.

```text
impl fn format<...As, ...Ks>{
    const not_valid: str = "<not valid>",
    const ticked: str = "*",
    const sample: i64 = 1,
    const print_delta: bool = true,
    const require_valid: bool = true,
    trigger: signal?,
}(const fmt_str: str, args: ...As, kwargs: ...{Ks}) -> str
```

## Rules

- **FMT-1.** A function that takes a format string takes it as a `const`
  `str`, then a positional pack and a named pack: the values its places are
  filled with. The rules below hold for every such function of core.
- **FMT-2.** A place is written in braces. `{}` takes the next positional
  argument, in the order given; `{name}` takes the named argument of that
  name. Either may be followed, inside the braces, by `:` and a specifier
  (FMT-4): `{:.2f}`, `{level:>3}`. `{{` and `}}` are a brace each. Nothing
  else is a place: no position is written as a number.
- **FMT-3.** The format string is fixed where the call is compiled, and is
  read there against the arguments the call gives. A call rejects where a
  brace opens no place and closes none (`format.syntax`); where a `{}` has
  no positional argument left, or a `{name}` names no named argument
  (`format.argument`); where an argument is taken by no place
  (`format.unused`); and where a specifier is one its argument's type does
  not admit (`format.specifier`). Nothing of a format string is found
  wrong while the program runs. A generic body may format an argument
  whose type depends on its generic parameters without a `requires`
  for that type. The specifier and text conversion are checked in the
  instance, at the call that makes it (GEN-7).
- **FMT-4.** A specifier is of the format language of the fmt library,
  which is Python's: fill and alignment, sign, width, precision, and type.
  Subject to this rule, `i64`, `f64`, `str` and `bool` admit the
  library's specifiers for integers, floating-point numbers, strings
  and booleans respectively. The date and time directives for `date`,
  `time`, `datetime` and `duration`, such as `{:%Y-%m-%d}`, are described
  below. A value of any other type takes `{}` alone. Width and precision
  of a `str` count its characters. A specifier that supplies neither a
  type letter nor a precision writes an `f64` as its FMT-5 text, and
  its sign, fill, alignment and width apply to that text: `{:+}` of
  `3.0` is `+3.0`. The `0` flag on a `bool` or a `str` rejects
  (`format.specifier`), as does the type letter `c` on an `i64`.
  The specifier `?` alone writes a `str` as a double-quoted HGL
  string literal, with the escapes FMT-5 uses inside a compound value.
  `%S` writes whole seconds, and so does `%T`, which ends in them. A
  precision after either writes the fraction of the second too, to the
  millisecond, the microsecond or the nanosecond, the digits after it
  dropped. It is taken out where
  the call is compiled, and the library is given the rest and the value
  at that precision. The precision is a `.` and one digit
  directly after the directive, `{:%H:%M:%S.3}` for milliseconds,
  `09:30:00.250`, `.6` for microseconds, `.9` for nanoseconds, and `.0`
  for none, as with no precision. A `time` admits the directives that
  read the hour, the minute and the second, `%H`, `%M`, `%S`, `%I`,
  `%p`, `%R` and `%T`, and a `date` those that read a date; one of the
  other's rejects (`format.specifier`). A `datetime` admits both.
  A `duration` admits those of a `time`, and its `%H` is the hours left
  after its whole days, `00` to `23`, as that library writes it. The
  whole of a `duration` as hours is asked for by a specifier of its
  own, so that `%H` means one thing and neither is taken for the other.
  That specifier is the unit's directive in square brackets, and the
  minutes and the seconds have theirs: `%[H]` the whole of it in hours,
  `%[M]` in minutes and `%[S]` in seconds, each of two digits or more,
  and `%[S]` with a precision as `%S` has: `{:%[H]:%M:%S}` of `1d1h` is
  `25:00:00`. It is taken out where the call is compiled, as a
  precision is. The directives of a date are those the reference
  lists. That library's locale flag, `L`, is not admitted, so that a
  text does not depend on where the program runs. `%j` writes a
  `duration`'s whole days. A fill, an alignment and a width stand
  before the directives, `{:>12%H:%M}`, and pad what they write; with
  no directive they pad the text FMT-5 gives, `{:>12}` of the time half
  past nine being `"    09:30:00"`. A sign, `#`, `0` or a type letter
  there rejects (`format.specifier`).
- **FMT-5.** With no specifier a `str` is written as its characters, with
  no quotes, and an `i64`, an `f64` and a `bool` as `convert<str>` of each
  gives ([scalar operators](scalar-operators.md), OPS-6). A `Type` is
  written by its native value `convert<str>` as its canonical spelling,
  the spelling a diagnostic writes: `list<i64, 2>`, `dict<str, f64>`,
  `i64?`, and a struct by its name (TYPE-18). A `date`, a
  `time` and a `datetime` with no specifier are written in their ISO 8601
  form: `2026-11-22`; `09:30:00`, and where
  the fraction of a second is not zero with its six digits,
  `09:30:00.250000`; and `2026-11-22T09:30:00Z`, a `datetime` being an
  instant in UTC, with its fraction as a `time` has it. A `duration` is
  written as its literal is, in its largest units, a whole number of
  each unit that is not zero: `1h30m`, `1s500ms`, and `0s` for none.
  A nullable argument is written as its value's text when it has a
  value and as `null` when it has none. A list, set or dictionary delta
  is written as its constructor (TYPE-16), omitting empty or null
  properties.
  Other values use their type's `convert<str>`, core's or a candidate
  the program declares for a type of its own. Where the type has no
  conversion, the call rejects where it is compiled.
  Core gives `convert<str>` for a list, a set, a dictionary, a tuple and
  a struct (OPS-4), which writes the value as a program writes it, each
  part by this rule and a `str` among them in quotes: `[1, 2]` for a
  list and for a set, `["a": 1]`, `(1, "a")`, and
  `Point(x: 1.0, y: 2.0)` with every field in the order declared.
  A `duration` below zero is written after a `-`; a set and a
  dictionary are written in the order a `for` over them visits; a part
  that is null is written `null`; and a `str` among the parts is written
  with the escapes its literal has.
- **FMT-6.** `format` is an operator with a candidate for value code and
  one for a graph. Called where a graph is built its result is a text
  that ticks: in each cycle in which an argument ticks, once every
  argument is valid. With no arguments it ticks once, when the graph
  starts.
- **FMT-7.** An argument may be a `Marked<T>`: a value of T or none, and
  whether it ticked. Its place admits the specifiers T admits (FMT-4).
  Where it holds a value the place is written as that value is. Where it
  holds none the place is written as the text of the `not_valid` option,
  `<not valid>` unless the call gives another, and its specifier is not
  applied.
- **FMT-8.** In a graph a call takes the options a line of
  `debug_print` is made with, since that is made by this
  ([diagnostics](diagnostics.md)). `require_valid`, which is true unless
  given, holds the text until every argument is valid; with
  `{require_valid: false}` it ticks before that, and an argument with no
  value is written as FMT-7 writes none. `print_delta` says whether a
  collection that ticked is written as its delta (FMT-5), which is the
  default, or whole. `sample` gives one text in every that many;
  it is 1 or more. With a `trigger`, a connection taken as a signal, the
  text ticks in each cycle in which the trigger ticks and in no other,
  with the arguments as they stand, and with no arguments at each tick of
  the trigger and not when the graph starts. The HGL graph candidate controls
  publication and passes its clock and its positional and named inputs to
  the native formatter. The formatter borrows each endpoint for that call
  (FN-8, TIME-9). Its payload type determines which specifiers are valid.
  The formatter reads the payload only if the endpoint is valid. A constant
  wired as a temporal input publishes at graph start (FN-3).
- **FMT-9.** The evaluation time, the time of day, and a mark on what
  ticked are asked for in the format string itself. Where the call is
  compiled they are taken out of it, and the rest is given to the fmt
  library with what they add. `{engine_time}` is the evaluation time of
  the cycle the text is made in, and `{wall_time}` the time of day of
  the machine; each is a `datetime`, takes a specifier as one does, and
  is no name a call gives an argument under. A `*` after a place's
  opening brace marks the place, `{*}`, `{*:.2f}`, `{*mid}`: it is
  written with the text of the `ticked` option, `*` unless the call
  gives another, before its value, where its argument ticked in the
  cycle. The prefix also precedes the `not_valid` text when a
  `Marked<T>` ticked and holds no value. With the defaults, `{*}`
  writes `*<not valid>` for that argument. The evaluation time and a
  mark are of a graph, where there is a cycle and an argument ticks:
  a call where a graph is built asks for
  them. `{wall_time}` is asked for anywhere. In value code a call that
  asks for `{engine_time}` rejects (`format.argument`), and so does one
  that marks a place whose argument is no `Marked` (`format.specifier`);
  a mark on a `Marked` is written there where its `ticked` is true.
  For a borrowed graph input, the formatter tests whether the endpoint
  was modified in the current cycle. It reads the supplied clock’s
  evaluation time once per call that requests `{engine_time}`.

## Reference

Everything a format string may hold, in one place. The rules above decide;
this section restates them with what each form writes.

### Places

| Written | Filled with |
| --- | --- |
| `{}` | The next positional argument |
| `{name}` | The named argument `name` |
| `{:spec}`, `{name:spec}` | The same, written as the specifier says |
| `{*}`, `{*name}`, `{*:spec}` | The same, after a `*` where the argument ticked in the cycle: in a graph, and in value code for a `Marked` |
| `{engine_time}`, `{engine_time:spec}` | The evaluation time of the cycle, a `datetime`: in a graph |
| `{wall_time}`, `{wall_time:spec}` | The time of day of the machine, a `datetime`: anywhere |
| `{{`, `}}` | A brace |

### A number, a string and a boolean

A specifier is `[[fill]align][sign][#][0][width][.precision][type]`: each
part may be left out, and those written stand in that order.

| Part | Written | Writes | Example |
| --- | --- | --- | --- |
| Alignment | `<`, `>` or `^` | The value at the left, the right or the middle of its width. A number is at the right unless told, anything else at the left | `{:>6}` of `"ab"` is `"    ab"` |
| Fill | Any character, before the alignment | That character where the spaces would be | `{:*^6}` of `"ab"` is `"**ab**"` |
| Sign | `+`, or a space | A `+`, or a space, before a number that is not below zero. An `f64` with neither a type letter nor a precision keeps its FMT-5 text | `{:+}` of `5` is `"+5"`, and of `3.0` is `"+3.0"` |
| Base prefix | `#` | `0x`, `0b` or `0` before a number in that base | `{:#x}` of `255` is `"0xff"` |
| Zeros | `0`, before the width | Zeros after the sign where the spaces would be; rejects for a `bool` or `str` | `{:05}` of `42` is `"00042"` |
| Width | A number | At least that many characters. An `f64` with neither a type letter nor a precision is padded as its FMT-5 text | `{:>4}` of `"日"` is `"   日"`; `{:>6}` of `3.0` is `"   3.0"`; `{:5}` of `true` is `"true "` and `{:>5}` is `" true"` |
| Precision | `.` and a number | Of an `f64`, the digits after the point (`f`, `e`) or the significant digits (`g`); of a `str`, the characters kept | `{:.2f}` of `101.5` is `"101.50"`; `{:.2}` of `"日日日"` is `"日日"` |

| Type | Of | Writes | Example |
| --- | --- | --- | --- |
| `d` | `i64`, `bool` | Decimal, which an `i64` has with none | `{:d}` of `true` is `"1"` |
| `x`, `X` | `i64` | Hexadecimal, in small or capital letters | `{:x}` of `255` is `"ff"` |
| `o` | `i64` | Octal | `{:o}` of `8` is `"10"` |
| `b` | `i64` | Binary | `{:b}` of `5` is `"101"` |
| `f` | `f64` | The digits after the point that the precision gives, six with none | `{:.2f}` of `3.14159` is `"3.14"` |
| `e`, `E` | `f64` | One digit, the point, and an exponent | `{:.2e}` of `1234.5` is `"1.23e+03"` |
| `g`, `G` | `f64` | The significant digits the precision gives, six with none, and no zeros after them: with an exponent where that exponent is below -4 or not below the number of digits, and without one otherwise | `{:g}` of `1234.5` is `"1234.5"`; of `100000.0`, `"100000"`; of `1000000.0`, `"1e+06"` |
| `s` | `str`, `bool` | The text, which each has with none | `{:s}` of `true` is `"true"` |
| `?` alone | `str` | The quoted HGL literal | The text of `{:?}` applied to a newline is `"\n"` |

These are the forms in common use. FMT-4 also admits the
[fmt library's format language](https://fmt.dev/latest/syntax/) for these
four types, subject to the restrictions in the rule.

### A date, a time and a duration

A specifier of these is made of directives, each a `%` and a letter, with
any text between them: `{:%Y-%m-%d %H:%M}`. A fill, an alignment and a
width may stand before them, as for a number, and pad the whole:
`{:>12%H:%M}` is `"       09:30"`. The examples are of
`2026-11-22T09:30:00.25Z`, a Sunday.

| Directive | Writes | Example | Taken by |
| --- | --- | --- | --- |
| `%Y` | The year | `2026` | `date`, `datetime` |
| `%y` | The last two digits of the year | `26` | `date`, `datetime` |
| `%m` | The month, `01` to `12` | `11` | `date`, `datetime` |
| `%b`, `%B` | The month's name, short and whole | `Nov`, `November` | `date`, `datetime` |
| `%d` | The day of the month, `01` to `31` | `22` | `date`, `datetime` |
| `%e` | The day of the month, a space before a single digit | `22` | `date`, `datetime` |
| `%a`, `%A` | The day of the week, short and whole | `Sun`, `Sunday` | `date`, `datetime` |
| `%j` | The day of the year, `001` to `366` | `326` | `date`, `datetime` |
| `%F` | `%Y-%m-%d` | `2026-11-22` | `date`, `datetime` |
| `%D` | `%m/%d/%y` | `11/22/26` | `date`, `datetime` |
| `%H` | The hour, `00` to `23` | `09` | `time`, `datetime`, `duration` |
| `%I` | The hour, `01` to `12` | `09` | `time`, `datetime`, `duration` |
| `%p` | `AM` or `PM` | `AM` | `time`, `datetime`, `duration` |
| `%M` | The minute, `00` to `59` | `30` | `time`, `datetime`, `duration` |
| `%S` | The second, `00` to `59`, whole | `00` | `time`, `datetime`, `duration` |
| `%S.3`, `%S.6`, `%S.9` | The second with its fraction: to the millisecond, the microsecond, the nanosecond | `00.250`, `00.250000`, `00.250000000` | `time`, `datetime`, `duration` |
| `%R` | `%H:%M` | `09:30` | `time`, `datetime`, `duration` |
| `%T` | `%H:%M:%S`, and `%T.3` as `%S.3` | `09:30:00`, `09:30:00.250` | `time`, `datetime`, `duration` |
| `%[H]`, `%[M]`, `%[S]` | The whole of a duration in hours, in minutes, in seconds: of `1d1h30m`, | `25`, `1530`, `91800` | `duration` |
| `%j` | Its whole days: of `1d1h30m`, | `1` | `duration` |
| `%%` | A percent sign | `%` | Each of them |

The names of a month and of a day of the week are the English ones.

Of a `duration`, `%H` is the hours left after its whole days, and
`%[H]` the whole of it in hours: of `1d1h`, `{:%H:%M:%S}` is `01:00:00`,
`{:%[H]:%M:%S}` is `25:00:00`, and `{:%j %T}` is `1 01:00:00`.

### With no specifier

| Type | Written | Example |
| --- | --- | --- |
| `str` | Its characters, with no quotes | `fill` |
| `i64` | Decimal | `42`, `-3` |
| `f64` | The shortest text that reads back as the number, always with a `.` or an exponent | `101.5`, `3.0`, `1e21` |
| `bool` | `true` or `false` | `true` |
| `date` | ISO 8601 | `2026-11-22` |
| `time` | ISO 8601, with the fraction of a second, six digits, only where it is not zero | `09:30:00`, `09:30:00.250000` |
| `datetime` | ISO 8601, in UTC, its fraction as a `time`'s | `2026-11-22T09:30:00Z` |
| `duration` | Its literal, in its largest units | `1h30m`, `1s500ms`, `0s`, `-250ms` |
| A list, a set | As a program writes it, a `str` among its parts in quotes | `[1, 2]`, `["a", "b"]` |
| A dictionary | The same | `["a": 1]` |
| A tuple | The same | `(1, "a")` |
| A struct | The same, every field in the order declared | `Point(x: 1.0, y: 2.0)` |
| A nullable argument | Its value's text, or `null` | `42`, `null` |
| A null among the parts of a collection, tuple or struct | `null` | `[1, null]` |
| A list delta | Its TYPE-16 constructor, omitting empty or null properties | `delta<list<i64>>(updated: [0: 4])` |
| A set delta | The same | `delta<set<i64>>(added: [1, 2], removed: [3])` |
| A dictionary delta | The same | `delta<dict<str, i64>>(updated: ["a": 1], removed: ["b"])` |
| A `Marked<T>` that holds none | The `not_valid` text, with the prefix for a marked place where it ticked | `<not valid>`, `*<not valid>` |
| Any other type | What `convert<str>` of it gives; the call rejects where the type has none | `1W6D` for a `tenor` |

### Options

| Option | Of | With none | Does |
| --- | --- | --- | --- |
| `not_valid: str` | `format` | `"<not valid>"` | The text written for an argument that has no value |
| `ticked: str` | `format` | `"*"` | The text written before a marked value that ticked |
| `require_valid: bool` | `format` and `debug_print`, in a graph | `true` for `format`, `false` for `debug_print` | Holds the text until every argument is valid |
| `print_delta: bool` | `format` and `debug_print`, in a graph | `true` | A collection that ticked is written as what changed in the cycle; with `false`, whole |
| `sample: i64` | `format` and `debug_print`, in a graph | `1` | One text in every that many |
| `trigger: signal` | `format` and `debug_print`, in a graph | None given | The text ticks when the trigger ticks, and at no other time |

### Rejections

Each is reported where the call is compiled.

| Code | When | Example |
| --- | --- | --- |
| `format.syntax` | A brace opens no place and closes none | `format("at {", price)` |
| `format.argument` | A place has no argument; `{engine_time}` in value code | `format("{} at {}", qty)` |
| `format.unused` | An argument no place takes | `format("fill", qty)` |
| `format.specifier` | A specifier its argument's type does not admit; in value code, a mark on an argument that is no `Marked` | `format("{:05}", true)`, `format("{:c}", 65)` or `format("{:%H}", day)` with a `date` |

## Examples

Accepted illustration, complete module:

```hgl
module examples.formatting

const fn filled(qty: i64, price: f64) -> str =>
    format("fill {} at {:.2f}", qty, price)

const fn padded(level: i64) -> str =>
    format("level {level:>3} of {{book}}", level: level)

const fn shown(ticked: bool, price: f64?) -> str =>
    format("mid {*:.1f}", Marked<f64>(value: price, ticked: ticked))

fn quoted(bid: f64, ask: f64) -> str =>
    format{require_valid: false}("{engine_time:%H:%M:%S} {*:.2f} / {*:.2f}", bid, ask)

test texts {
    assert filled(10, 101.5) == "fill 10 at 101.50"
    assert padded(7) == "level   7 of {book}"
    assert shown(true, 101.5) == "mid *101.5"
    assert shown(false, 101.5) == "mid 101.5"
    assert shown(false, null) == "mid <not valid>"
}
```

`quoted` is a text that ticks whenever `bid` or `ask` does: with `bid`
ticking at 100.0 and `ask` not yet valid, at half past nine,
`09:30:00 *100.00 / <not valid>` (FMT-8, FMT-9).
Deliberate rejections, each where its call is compiled:
`format("fill {} at {}", qty)`, a place with no argument left
(`format.argument`); `format("fill {qty}", quantity: qty)`, a name no
argument has (`format.argument`); `format("fill", qty)`, an argument no
place takes (`format.unused`); `format("at {:.2f}", qty)` with `qty` an
`i64`, a precision an integer does not admit (`format.specifier`); and
`format("at {", price)`, a brace that opens no place (`format.syntax`).
The [Core conformance cases](../tests/core_conformance/formatting.hgl) exercise
the value candidate. The catalogue codes have their own
[source rejection cases](../tests/conformance/rejections/formatting.hgl).
