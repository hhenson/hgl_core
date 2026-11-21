# Diagnostics

Status: DBG-1 through DBG-4 are agreed. DBG-5 through DBG-7 are proposed.
Nothing here is declared in `hgl/` yet.

## Scope and API

`debug_print` is a tool for the person writing a program: it shows what is
happening while the program runs. It is meant to be put in, looked at, and
taken out again, and a project may well refuse it in code it keeps.
Logging that stays is the logger's ([capabilities](capabilities.md)).

It takes a format string and the values to fill it with
([formatting](formatting.md)).

| Core API | Contract |
| --- | --- |
| `debug_print{options}(fmt_str, args..., kwargs...)` | One line on standard output, written when it is called |

```hgl
export operator debug_print<...As, ...Ks>(
    const fmt_str: str, args: ...As, kwargs: ...{Ks})

# In a graph: a line whenever an argument ticks.
native impl fn debug_print<...As, ...Ks>{
    const sample: i64 = 1,
    const print_delta: bool = true,
    const engine_time: bool = false,
    const wall_time: bool = false,
    const require_valid: bool = false,
    const mark_ticked: bool = false,
}(const fmt_str: str, args: ...As, kwargs: ...{Ks})

# In a node's handler or a value function: a line where it is executed.
native impl const fn debug_print<...As, ...Ks>(
    const fmt_str: str, args: ...As, kwargs: ...{Ks})
```

## Rules

- **DBG-1.** `debug_print` writes one line to the process's standard
  output, the format string with its places filled (FMT-1 to FMT-5), and
  gives no result. The line is written when the call is evaluated, to the
  stream itself: it is not held to the end of the cycle or of the run, and
  it is no part of a run's record.
- **DBG-2.** Called where a graph is built, it writes a line in each cycle
  in which any of its arguments ticks. With no arguments it writes its line
  once, when the graph starts.
- **DBG-3.** Called in a node's handler or in a value function, it writes
  its line where it is executed, as a print does in any language, with the
  values its arguments have there.
- **DBG-4.** In a graph a call takes options. `engine_time` puts the
  cycle's evaluation time before the text, and `wall_time` the time of day
  of the machine; both are off unless asked for. `sample` writes one line
  in every that many: `{sample: 10}` the tenth, the twentieth, and so on;
  it is 1 or more. `print_delta` says whether a collection that ticked is
  written as what changed in the cycle, which is the default, or whole.
- **DBG-5.** In a graph an argument may have no value when another ticks.
  It is written as `<not valid>`. With `{require_valid: true}` no line is
  written until every argument is valid.
- **DBG-6.** With `{mark_ticked: true}` each value that ticked in the cycle
  is written with `*` before it, so that what set the line off is seen.
- **DBG-7.** It changes nothing else a program can observe. Where several
  calls write in one cycle, their lines stand in the order their nodes are
  evaluated, which the language orders by what depends on what and not
  otherwise.

## Examples

Accepted illustration, complete module:

```hgl
module examples.diagnostics

fn mid(bid: f64, ask: f64) -> f64 {
    debug_print("quote {:.2f} / {:.2f}", bid, ask)
    let value = (bid + ask) / 2.0
    debug_print{engine_time: true, mark_ticked: true}("mid {value:.3f}", value: value)
    return value
}

fn counted(value: i64) -> i64 {
    state seen: i64 = 0
    when {
        seen += 1
        debug_print("tick {} is {}", seen, value)
        return seen
    }
}
```

In `mid`, the first call writes a line whenever `bid` or `ask` ticks: with
`bid` at `100.0` and `ask` not yet valid, `quote 100.00 / <not valid>`
(DBG-2, DBG-5). The second writes `value` once both are valid, with the
evaluation time before it and a `*` where it ticked (DBG-4, DBG-6). In
`counted` the call is in a handler, and writes its line each time the
handler runs it (DBG-3). No conformance case observes the lines: a trace
does not see standard output.
