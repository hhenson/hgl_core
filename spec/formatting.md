# Formatting

Status: FMT-1 through FMT-4 are agreed. FMT-5 and FMT-6 are proposed, and
so are the diagnostic codes FMT-3 names. Nothing here is declared in
`hgl/` yet.

## Scope and API

A format string is text with places for values. One rule serves every
function of core that takes one: `format`, which gives the text, and
`debug_print` ([diagnostics](diagnostics.md)), which writes it. The language
specification says what a pack is and how a call gives one
([functions](https://github.com/hhenson/hgl_spec/blob/main/spec/language/functions-and-phases.md),
FN-8).

| Core API | Contract |
| --- | --- |
| `format(fmt_str, args..., kwargs...)` | The text of `fmt_str` with its places filled |

```hgl
export native const fn format<...As, ...Ks>(
    const fmt_str: str, args: ...As, kwargs: ...{Ks}) -> str
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
- **FMT-6.** `format` is a value function. Called where a graph is built
  with connections among its arguments it is lifted as any value function
  is (the language's FN-6): its result ticks when an argument ticks, once
  every argument is valid.

## Examples

Accepted illustration, complete module:

```hgl
module examples.formatting

const fn filled(qty: i64, price: f64) -> str =>
    format("fill {} at {:.2f}", qty, price)

const fn padded(level: i64) -> str =>
    format("level {level:>3} of {{book}}", level: level)

test texts {
    assert filled(10, 101.5) == "fill 10 at 101.50"
    assert padded(7) == "level   7 of {book}"
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
