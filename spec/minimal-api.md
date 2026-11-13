# Minimal core API

Status: proposed capability inventory. No native signatures are accepted yet.

Each row must gain a contract, examples, and conformance observations before it
becomes mandatory. API names and representation choices remain open.
The [numeric operator contract](numeric-operators.md) now defines the agreed
numeric baseline and policy behavior; other inventory rows retain their status.
The next proposed slice is [scalar endpoint observations](temporal-endpoints.md),
separating publication, validity, notification, and retained data.

| Capability | HGL-facing need | Implementation-facing obligation |
| --- | --- | --- |
| Values and types | Typed ordinary values and capability-constrained operations | Preserve type identity, value ownership, and admitted operations |
| Temporal endpoints | Read values and publication metadata; publish outputs | Preserve validity, deltas, equal ticks, and modification times |
| Admission and activity | React to admitted input changes; select input activity | Enforce subscriptions and validity guards without inventing ticks |
| Time and execution | Evaluation time and ordered temporal computation | Define run bounds, cycle order, idle completion, and failure propagation |
| Graph construction | Compose functions and bind typed connections | Validate descriptions and instantiate independent graphs |
| Lifecycle and state | Start/stop hooks, persistent node state, reconstructible cache | Scope storage, initialization, teardown, and failure cleanup |
| Scheduling and sources | Future publications and source suspension | Schedule, cancel where admitted, and resume in the defined time order |
| Runtime capabilities | Clock, output access, logging, and run-wide keyed state | Expose only declared capabilities with defined lifetimes and access rules |
| Dynamic structure | References, collection membership, and child graphs | Preserve identity, rebinding, child lifetime, and publication order |
| Core operations | Syntax-bound operations and essential named functions | Supply baseline behavior over explicitly required types and phases |
| Test execution | Eval, replay, recording, and observable outcomes | Isolate runs, retain captures, and report failures without masking them |

## Proposed first slice

Start with scalar input/output, a compute fixture, finite replay, recording,
and independent run lifetime. Use `i64`, `bool`, and `str` to expose zero,
false, and empty-text publications. This is an exercise profile, not a claim
that the remaining inventory is optional for eventual HGL conformance.
