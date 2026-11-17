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

[`tests/conformance/`](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/README.md)
holds the endpoint and eval fixtures. `hgl/core/core.hgl` is the mandatory
module, `core`: the comparison outcome, the option enums, the run
configuration and the symbol operators, with HGL wrappers for commutative
pairs in reverse operand order.
Generic temporal candidates handle same-type addition/subtraction with an
explicit `invalid` input policy (OPS-9).
`hgl/native/scalar_operators.hgl` holds the remaining native scalar candidates,
each declaring the options it uses. `hgl/native/capabilities.hgl` holds the
capability types, logging, scheduling, and the global store's candidates,
and `hgl/native/endpoints.hgl` the endpoint operations. Providers implement only
those native declarations. [Implementation tests](https://github.com/hhenson/hgl_core/blob/main/tests/implementation/README.md) cover
the wrappers.
A target adapter and eval runner are needed to run the shared expectations.
