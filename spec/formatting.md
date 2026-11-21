# Formatting

Status: FMT-1 through FMT-4 and FMT-7 are agreed; the names FMT-7 gives
`Marked` and the two options are proposed. FMT-5, FMT-6 and FMT-8 are
proposed, and so are the diagnostic codes FMT-3 names. Nothing here is
declared in `hgl/` yet.

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
    const engine_time: bool = false,
    const wall_time: bool = false,
    const require_valid: bool = true,
    const mark_ticked: bool = false,
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
  The scalar types admit one: `i64`, `f64`, `str` and `bool`, each with the
  specifiers that library admits for an integer, a floating-point number, a
  string and a boolean. A value of any other type takes `{}` alone.
- **FMT-5.** With no specifier a value is written as its text: a `str` as
  its characters, with no quotes; an `i64`, an `f64` and a `bool` as
  `str` of each gives; any other value in a form that shows its parts, the
  strings among them in quotes, for a person to read.
- **FMT-6.** `format` is an operator with a candidate for value code and
  one for a graph. Called where a graph is built its result is a text
  that ticks: in each cycle in which an argument ticks, once every
  argument is valid. With no arguments it ticks once, when the graph
  starts.
- **FMT-7.** An argument may be a `Marked<T>`: a value of T or none, and
  whether it ticked. Its place admits the specifiers T admits (FMT-4).
  Where it holds a value the place is written as that value is, with the
  text of the `ticked` option before it where `ticked` is true. Where it
  holds none the place is written as the text of the `not_valid` option,
  and its specifier is not applied. The two options are `format`'s own
  and are given in its braces; left out, they are `*` and `<not valid>`.
- **FMT-8.** In a graph a call takes the options a line of
  `debug_print` is made with, since that is made by this
  ([diagnostics](diagnostics.md)). `require_valid`, which is true unless
  given, holds the text until every argument is valid; with
  `{require_valid: false}` it ticks before that, and an argument with no
  value is written as FMT-7 writes none. `mark_ticked` writes each
  argument that ticked in the cycle as FMT-7 writes one that ticked.
  `print_delta` says whether a collection that ticked is written as what
  changed in the cycle, which is the default, or whole. `engine_time`
  puts the cycle's evaluation time before the text, and `wall_time` the
  time of day of the machine. `sample` gives one text in every that many;
  it is 1 or more. With a `trigger`, a connection taken as a signal, the
  text ticks in each cycle in which the trigger ticks and in no other,
  with the arguments as they stand, and with no arguments at each tick of
  the trigger and not when the graph starts. The candidate for a graph is
  written in HGL: it gives the native each argument as a `Marked`.

## Examples

Accepted illustration, complete module:

```hgl
module examples.formatting

const fn filled(qty: i64, price: f64) -> str =>
    format("fill {} at {:.2f}", qty, price)

const fn padded(level: i64) -> str =>
    format("level {level:>3} of {{book}}", level: level)

const fn shown(price: f64?, ticked: bool) -> str =>
    format("mid {:.1f}", Marked<f64>(value: price, ticked: ticked))

test texts {
    assert filled(10, 101.5) == "fill 10 at 101.50"
    assert padded(7) == "level   7 of {book}"
    assert shown(101.5, true) == "mid *101.5"
    assert shown(101.5, false) == "mid 101.5"
    assert shown(null, false) == "mid <not valid>"
}
```

Deliberate rejections, each where its call is compiled:
`format("fill {} at {}", qty)`, a place with no argument left
(`format.argument`); `format("fill {qty}", quantity: qty)`, a name no
argument has (`format.argument`); `format("fill", qty)`, an argument no
place takes (`format.unused`); `format("at {:.2f}", qty)` with `qty` an
`i64`, a precision an integer does not admit (`format.specifier`); and
`format("at {", price)`, a brace that opens no place (`format.syntax`).
No conformance case is claimed.
