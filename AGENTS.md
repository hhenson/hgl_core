# Core rules

- Before every push, check the branch's PR state on GitHub. If its previous PR
  is merged, fetch `origin/main`, start a fresh `codex/` branch from it, and carry
  forward only the new changes. Do not push follow-up commits to a merged PR's
  branch; open a new PR for the fresh branch.
- Keep documents and discussions terse. Avoid extended dialogue.
- Read the [foundation](spec/foundation.md). Label agreed and proposed text.
  Keep open questions and decision checklists in this repository's issue tracker,
  outside specification and explanation documents. Do not turn proposed API rows
  into requirements without a decision.
- Use `spec/` for rules, examples/APIs, and useful diagrams. Add explanations
  under `explanations/` only when a concept needs detailed justification.
- Put runtime native declarations in `hgl/native/`; core HGL bodies and required
  native implementation parts in `hgl/core/`. Add target source only as required.
- Keep portable conformance in `tests/conformance/`; ordinary implementation
  tests in `tests/implementation/`. Do not claim execution for a document example.
- Define observable obligations before target symbols or internal layouts.
  Native adapters must preserve shared expected results.
- Core has no optional-library dependency. Keep the test host's privileges
  separate from the API available to ordinary HGL programs.
- Use the [language specification](https://github.com/hhenson/hgl_spec) directly.
  Require no historical repository or experiment record to understand core.
- Report the exact tests run, skipped scope, and unresolved capabilities.
