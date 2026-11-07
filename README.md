# HGL core

The mandatory API and runtime contract for HGL. Core owns its declarations,
portable implementations, required native support, and runtime conformance.

- [Foundation charter](spec/foundation.md)
- [Minimal API inventory](spec/minimal-api.md)
- [Numeric operators and policies](spec/numeric-operators.md)
- [Value-function lifting observations](spec/value-function-lifting.md)
- [Scalar temporal endpoints (proposed)](spec/temporal-endpoints.md)
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

Code directories are reserved; practical examples will populate them. This
foundation contains documents only and makes no conformance claim. A target
supplies an adapter and runs the shared expectations unchanged.
