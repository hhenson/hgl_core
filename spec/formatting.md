# Formatting

Status: FMT-1 through FMT-9 are agreed. Nothing here is declared in `hgl/`
yet.

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

```hgl
export operator format<...As, ...Ks>(
    const fmt_str: str, args: ...As, kwargs: ...{Ks}) -> str

# In a node's handler or a value function: the one native.
native impl const fn format<...As, ...Ks>{
    const not_valid: str = "<not valid>",
    const ticked: str = "*",
}(const fmt_str: str, args: ...As, kwargs: ...{Ks}) -> str

# In a graph: its text ticks. Written in HGL on the native.
impl fn format<...As, ...Ks>{
    const not_valid: str = "<not valid>",
    const ticked: str = "*",
    const sample: i64 = 1,
    const print_delta: bool = true,
    const require_valid: bool = true,
    trigger: signal?,
}(const fmt_str: str, args: ...As, kwargs: ...{Ks}) -> str

export struct Marked<T> {
    value: T?
    ticked: bool
}
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
  wrong while the program runs.
- **FMT-4.** A specifier is of the format language of the fmt library,
  which is Python's: fill and alignment, sign, width, precision, and type.
  The types that library formats admit one, each given to it as its own
  type with the specifier as written: `i64`, `f64`, `str` and `bool`, with
  the specifiers it admits for an integer, a floating-point number, a
  string and a boolean, and `date`, `time`, `datetime` and `duration`,
  with those it admits for a date and a time, `{:%Y-%m-%d}`. A value of
  any other type takes `{}` alone.
- **FMT-5.** With no specifier a `str` is written as its characters, with
  no quotes, and an `i64`, an `f64` and a `bool` as `convert<str>` of each
  gives ([scalar operators](scalar-operators.md), OPS-6). A value of a type
  FMT-4 does not name is written as `convert<str>` of it gives: the type
  has that conversion, core's or a candidate the program declares for a
  type of its own, and one that has none rejects where the call is
  compiled. A `date`, a `time` and a `datetime` with no specifier are
  written in their ISO 8601 form, and a `duration` as its literal is.
  Core gives `convert<str>` for a list, a set, a dictionary, a tuple and
  a struct, in a form that shows its parts, the strings among them in
  quotes, for a person to read (OPS-4).
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
  collection that ticked is written as what changed in the cycle, which
  is the default, or whole. `sample` gives one text in every that many;
  it is 1 or more. With a `trigger`, a connection taken as a signal, the
  text ticks in each cycle in which the trigger ticks and in no other,
  with the arguments as they stand, and with no arguments at each tick of
  the trigger and not when the graph starts. The candidate for a graph is
  written in HGL: it gives the native each argument as a `Marked`.
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
  cycle. The evaluation time and a mark are of a graph, where there is a
  cycle and an argument ticks: a call where a graph is built asks for
  them. `{wall_time}` is asked for anywhere. In value code a call that
  asks for `{engine_time}` rejects (`format.argument`), and so does one
  that marks a place whose argument is no `Marked` (`format.specifier`);
  a mark on a `Marked` is written there where its `ticked` is true,
  which is how the candidate for a graph gives the native what ticked
  (FMT-8).

## Examples

Accepted illustration, complete module:

```hgl
module examples.formatting

const fn filled(qty: i64, price: f64) -> str =>
    format("fill {} at {:.2f}", qty, price)

const fn padded(level: i64) -> str =>
    format("level {level:>3} of {{book}}", level: level)

const fn shown(price: f64?, ticked: bool) -> str =>
    format("mid {*:.1f}", Marked<f64>(value: price, ticked: ticked))

fn quoted(bid: f64, ask: f64) -> str =>
    format{require_valid: false}("{engine_time:%H:%M:%S} {*:.2f} / {*:.2f}", bid, ask)

test texts {
    assert filled(10, 101.5) == "fill 10 at 101.50"
    assert padded(7) == "level   7 of {book}"
    assert shown(101.5, true) == "mid *101.5"
    assert shown(101.5, false) == "mid 101.5"
    assert shown(null, false) == "mid <not valid>"
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
No conformance case is claimed.
