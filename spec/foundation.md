# Core foundation

Date: 2026-11-01. Status: agreed ownership; API details remain proposed.

## Agreed

- **CORE-1 — Purpose.** Specify the minimum API and runtime behavior required
  to support HGL. A target implements this contract independently of its layout.
- **CORE-2 — Interfaces.** Distinguish operations available to HGL code from the
  facilities required of a compiler/runtime adapter. Native declarations state
  that boundary explicitly.
- **CORE-3 — Ownership.** Core owns its API specification, HGL declarations and
  bodies, required native implementation, and all runtime conformance
  expectations. The language specification owns syntax and runtime concepts.
- **CORE-4 — Tests.** Share runtime conformance cases and expected observations.
  Target adapters supply missing host access without rewriting expectations.
  Ordinary tests accompany implementation code.
- **CORE-5 — Resources.** Specifications contain rules, example code/APIs, and
  diagrams. Add explanations only as needed. Code comprises HGL native
  declarations, core bodies, required native implementation, and tests.
- **CORE-6 — Closure.** Core and its conformance runs require no optional library.
  A selected native provider is part of core's declared implementation closure.
- **CORE-7 — Independence.** Core stands alone with the
  [language specification](https://github.com/hhenson/hgl_spec). Rules require
  no historical implementation or experiment record. Keep writing terse.

## Proposed next scope

Use the [minimal API inventory](minimal-api.md) to implement one
[dense scalar eval slice](testing-and-eval.md). Choose public names and native
signatures only as practical examples establish their necessity.

## Open

- Exact mandatory types, operations, native signatures, and capability phases.
- Portable bodies versus facilities that every target must supply natively.
- Adapter protocol and versioning of the API and conformance pack.
- Which host observations need a test-only interface.
