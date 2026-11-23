# Output

Status: OUT-1 and OUT-2 are agreed. The native is declared in
[`hgl/native/output.hgl`](../hgl/native/output.hgl).

## Scope and API

`write` puts text on one of the process's two output streams. It is the
one native that does: `debug_print` ([diagnostics](diagnostics.md)) is
written in HGL on it and on `format` ([formatting](formatting.md)).

| Core API | Contract |
| --- | --- |
| `write(text, stream, flush)` | The text, on standard output or standard error |

```hgl
export enum Stream { out, err }

export native const fn write(text: str, stream: Stream = .out, flush: bool = false)
```

## Rules

- **OUT-1.** `write` writes the characters of `text` to the process's
  standard output, or with `.err` to its standard error, as they are: it
  adds no line end. It gives no result. It writes where it is executed,
  in a node's handler or in a value function, and what it writes is the
  program's own output: no part of a run's record.
- **OUT-2.** With `flush: true` the stream is flushed before `write`
  returns, so that the text is seen then. Without it the stream decides
  when.

## Examples

Accepted illustration, complete module:

```hgl
module examples.output

fn counted(value: i64) -> i64 {
    state seen: i64 = 0
    when {
        seen += 1
        write(format("tick {}\n", seen), flush: true)
        write("counted\n", .err)
        return seen
    }
}
```

Each time the handler runs it writes a line to standard output, which is
flushed, and one to standard error. The
[Core conformance cases](../tests/core_conformance/output.hgl) observe the
counts and give the expected output on each stream.
