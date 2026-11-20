# Runtime conformance

Each named HGL test describes an observable runtime rule. The same source,
stimulus and expected result apply to every backend. Build fixtures from
ordinary HGL and the public operations of the mandatory core. Core operator
semantics have their own [implementation tests](../implementation/README.md).

## Reading a case

Start with the documentation comment immediately above a test, then read
its assertion. In a dense trace each column is one simulation step:
`_` means silence, a repeated value means a new publication, and `null`
means a published null. An invalidation is an action on an endpoint.
A subscriber can wake for that action even though the endpoint has no
value to forward.

A separate probe input lets a test inspect an endpoint on an idle cycle.
To observe which cycles woke a subscriber, have the subscriber publish a
marker and leave the probe out of its subscriptions.

## Where to start

| Runtime behavior | Example |
| --- | --- |
| Validity, silence and invalidation | [Scalar endpoint observers](scalar_endpoints.hgl) |
| Notifications and admission | [Input activity](input_activity.hgl) and [handler order](handler_order.hgl) |
| Fan-in and logical time | [Graph cycles](graph_cycles.hgl) |
| Node lifetime and shared state | [Run lifecycle](run_lifecycle.hgl) and [run-wide store](run_store.hgl) |
| Lifted values, fields and collection updates | [Lifted inputs](lifted_inputs.hgl) and [collection ticks](collection_ticks.hgl) |
| Window readiness | [Window ticks](window_ticks.hgl) |
| Branch selection and failure propagation | [Conditional graphs](conditional_graphs.hgl) and [captured failures](captured_failures.hgl) |
| Wakes and source suspension | [Scheduling](scheduled_wakes.hgl) and [source timing](source_timing.hgl) |
| References | [Reference ticks](reference_ticks.hgl) |
| Source rejections | [Collection children](rejections/collection_children.hgl), [references](rejections/references.hgl), [generic instances](rejections/generic_instances.hgl) and [scheduled wakes](rejections/scheduled_wakes.hgl) |

## Writing a case

Use an HGL `/** ... */` documentation comment attached to each named test.
Its first sentence states the observable rule. Explain a fixture or a
deliberate gap in the input when that helps a reader predict the outcome.
End the comment with two reStructuredText fields on separate lines:

- `:Rules:` lists the applicable core or language rule IDs, separated by commas.
- `:Status:` is `agreed` or `proposed`. If a case depends on a proposed
  profile, use `proposed` and name that dependency in the comment.

The module and test name identify the case. A documentation tool can use
the attached comment and the HGL assertion together. Keep the literal
expected trace in the test so a change to a tick is visible in review.
When existing backend tests describe the same observable behavior, choose
one small example that distinguishes it.

A source rejection is a declaration of the `conformance.rejections` module,
in `rejections/`. The language's `# expect-error(category, "code")`
annotation stands alone on the line before the rejected one (TST-12), and
the documentation comment is attached to the declaration. `hgl test`
matches the rejection; other commands read the annotation as a comment
(TST-14).

## Running the cases

From the core checkout, give the runner this checkout's core sources and
the conformance module. For example, with the hgraph target installed:

```sh
hgl check hgl tests/conformance
hgl test hgl tests/conformance --target hgraph
```

To run one case with its fixtures available, keep both source roots and add
`--test conformance::a_diamond_settles_before_its_join_runs`.

Report the core revision, compiler, backend and counts of cases passed,
failed and not run. A missing target capability leaves the expectation
in place. A backend failure can be narrowed down in its owning project;
compiler and transpiler regression tests belong with the affected code.
