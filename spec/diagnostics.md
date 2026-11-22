# Diagnostics

Status: DBG-1 through DBG-9 are agreed. Nothing here is declared in `hgl/`
yet.

## Scope and API

`debug_print` is a tool for the person writing a program: it shows what is
happening while the program runs. It is meant to be put in, looked at, and
taken out again, and a project may well refuse it in code it keeps.
Logging that stays is the logger's ([capabilities](capabilities.md)).

It takes a format string and the values to fill it with
([formatting](formatting.md)), and writes with `write`
([output](output.md)).

| Core API | Contract |
| --- | --- |
| `debug_print{options}(fmt_str, args..., kwargs...)` | One line on standard output, written when it is called |

```hgl
export operator debug_print<...As, ...Ks>(
    const fmt_str: str, args: ...As, kwargs: ...{Ks})

# In a graph: a line whenever an argument ticks, or the trigger.
impl fn debug_print<...As, ...Ks>{
    const sample: i64 = 1,
    const print_delta: bool = true,
    const require_valid: bool = false,
    trigger: signal?,
}(const fmt_str: str, args: ...As, kwargs: ...{Ks})

# In a node's handler or a value function: a line where it is executed.
impl const fn debug_print<...As, ...Ks>(
    const fmt_str: str, args: ...As, kwargs: ...{Ks})
{
    write(format(fmt_str, ...args, ...kwargs) + "\n", flush: true)
}
```

## Rules

- **DBG-1.** `debug_print` writes one line to the process's standard
  output, the format string with its places filled (FMT-1 to FMT-9), and
  gives no result. The line is written when the call is evaluated, to the
  stream itself: it is not held to the end of the cycle or of the run, and
  it is no part of a run's record.
- **DBG-2.** Called where a graph is built, with no trigger (DBG-8), it
  writes a line in each cycle in which any of its arguments ticks. With no
  arguments it writes its line once, when the graph starts.
- **DBG-3.** Called in a node's handler or in a value function, it writes
  its line where it is executed, as a print does in any language, with the
  values its arguments have there. Its format string asks for
  `{wall_time}` there, and for neither `{engine_time}` nor a mark, which
  are of a graph (FMT-9).
- **DBG-4.** In a graph a call takes options. `sample` writes one line in
  every that many: `{sample: 10}` the tenth, the twentieth, and so on; it
  is 1 or more. `print_delta` says whether a collection that ticked is
  written as what changed in the cycle, which is the default, or whole.
  The cycle's evaluation time and the time of day of the machine are no
  options: the format string asks for them, `{engine_time}` and
  `{wall_time}` (FMT-9).
- **DBG-5.** In a graph an argument may have no value when another ticks.
  It is written as `<not valid>` (FMT-7). With `{require_valid: true}` no
  line is written until every argument is valid.
- **DBG-6.** A place the format string marks, `{*}`, is written with `*`
  before its value where its argument ticked in the cycle (FMT-9), so
  that what set the line off is seen.
- **DBG-7.** It changes nothing else a program can observe. Where several
  calls write in one cycle, their lines stand in the order their nodes are
  evaluated, which the language orders by what depends on what and not
  otherwise.
- **DBG-8.** In a graph a call may give a trigger, `{trigger: beat}`: a
  connection taken as a signal. A line is then written in each cycle in
  which the trigger ticks, and in no other: a tick of an argument alone
  writes none, and the arguments are written as they stand. With no
  arguments a line is written at each tick of the trigger, and none when
  the graph starts.
- **DBG-9.** `debug_print` is no native. Both its candidates are written
  in HGL, in core, on `format`, which gives the line, and `write`, which
  puts it on the stream and flushes it. Its options in a graph are
  `format`'s there (FMT-8), given on to it, and the line is that text,
  written as it ticks; what differs is that `require_valid` is false
  unless given, so that a line shows what has no value yet.

## Examples

Accepted illustration, complete module:

```hgl
module examples.diagnostics

fn mid(bid: f64, ask: f64) -> f64 {
    debug_print("quote {:.2f} / {:.2f}", bid, ask)
    let value = (bid + ask) / 2.0
    debug_print("{engine_time} mid {*value:.3f}", value: value)
    return value
}

fn watched(book: dict<str, f64>, beat: signal) -> dict<str, f64> {
    debug_print{trigger: beat, print_delta: false}("book {}", book)
    return book
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
`watched` a line is written when `beat` ticks and at no other time, with
the book whole as it stands then (DBG-8). In `counted` the call is in a
handler, and writes its line each time the handler runs it (DBG-3). No
conformance case observes the lines: a trace does not see standard
output.
