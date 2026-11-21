# HGL core

The mandatory API and runtime contract for HGL. Core owns its declarations,
portable implementations, required native support, and runtime conformance.

- [Foundation charter](spec/foundation.md)
- [Minimal API inventory](spec/minimal-api.md)
- [Numeric operators and policies](spec/numeric-operators.md)
- [Scalar operators and conversion](spec/scalar-operators.md)
- [Temporal addition and subtraction](spec/temporal-arithmetic.md)
- [Value-function lifting observations](spec/value-function-lifting.md)
- [Scalar temporal endpoints](spec/temporal-endpoints.md)
- [Capabilities](spec/capabilities.md)
- [List operations](spec/list-operations.md)
- [Mapping and reduction](spec/mapping.md)
- [Output](spec/output.md)
- [Formatting](spec/formatting.md)
- [Diagnostics](spec/diagnostics.md)
- [Tenors and calendars](spec/tenor-operations.md)
- [Execution](spec/execution.md)
- [Testing and eval contract](spec/testing-and-eval.md)
- [Language specification](https://github.com/hhenson/hgl_spec)
- [Open decisions](https://github.com/hhenson/hgl_core/issues/1)

## Resources

| Path | Purpose |
| --- | --- |
| `spec/` | Rules, examples/APIs, and useful diagrams |
| `explanations/` | Justification added only when needed |
| `hgl/native/` | Runtime native declarations in HGL |
| `hgl/core/` | Core HGL bodies and required native implementation parts |
| `native/<target>/` | Target source only where a native implementation requires it |
| `tests/conformance/` | Shared runtime cases, fixtures, and expected results |
| `tests/implementation/` | Ordinary tests of core implementation code |

[The runtime suite](tests/conformance/README.md) documents observable behavior
with HGL programs and tick traces. `hgl/core/core.hgl` is the mandatory
module, `core`: the comparison outcome, the option enums, the run's
named times and the symbol operators, with HGL wrappers for commutative
pairs in reverse operand order.
Generic temporal candidates handle same-type addition/subtraction with an
explicit `invalid` input policy (OPS-9).
`hgl/native/scalar_operators.hgl` holds the remaining native scalar candidates,
each declaring the options it uses. `hgl/native/capabilities.hgl` holds the
capability types, the run's configuration among them, logging,
scheduling, and the global store's candidates,
`hgl/native/endpoints.hgl` the endpoint operations,
`hgl/native/lists.hgl` the operations that give a list more entries, and
`hgl/native/tenors.hgl` a tenor's arithmetic, order, and text.
`hgl/core/calendar.hgl` has the calendar and rolling-rule operators and, as
HGL, the candidates that move a date by a tenor.
Providers implement only those native declarations. `replay` and `record`,
which an eval's run is wired with, are HGL in `hgl/core/core.hgl`. [Implementation tests](https://github.com/hhenson/hgl_core/blob/main/tests/implementation/README.md) cover
scalar wrappers, identity arithmetic, list operations and tenors.
A target adapter and eval runner are needed to run the shared expectations.
