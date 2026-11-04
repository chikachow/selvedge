# Validation assurance and its cost

Research date: 2026-11-01. This informs the [design discussion](../design.md);
it is a recommendation, not an accepted release contract or implementation plan.
Complexity ratings below are engineering judgments about this project, not
measured effort or delivery estimates. Source inspection was performed; no
composition, compatibility, query-planning, or runtime experiments were run.

The worthwhile initial bundle is composition, public API change classification,
and visible Federation change review. Dangerous-change classification belongs in
that bundle because it comes from the same diff as breaking changes. Add
validation of explicitly supplied operation documents without building an
operation collection service. A claim that all five checks are mandatory would
instead require an owned operation corpus before release. These alternatives
need a product decision; an absent corpus must never appear as a successful
operation check.

## Relative cost and marginal assurance

The estimates assume reuse of Apollo composition and GraphQL Inspector, a Rust
CLI, the already accepted narrow Node adapter, complete local baseline/candidate
inputs, and an existing pull-request review interface. Implementing composition
or a compatibility classifier from scratch would be a materially different
undertaking.

| Check | Initial implementation cost | Continuing operation and test cost | Additional assurance and overlap |
| --- | --- | --- | --- |
| Composition, including satisfiability | Medium foundational work: input assembly, engine integration, diagnostics, API-schema extraction, and version compatibility. Engine selection remains an explicit dependency risk. | Medium: pin engine/options, maintain valid and invalid federation fixtures, and verify upgrades against the target router. No persistent service is inherently needed. | Establishes that the proposed subgraphs form a valid, statically satisfiable graph under that engine. It does not compare with the old public API. |
| Breaking public API classification | Small to medium after composition: reuse its two API schemas, call Inspector, preserve structured results, and map policy outcomes. | Low to medium: representative compatibility fixtures, dependency upgrades, and maintenance of the external repository rules governing exceptions. | Detects structural incompatibilities regardless of whether an affected client operation happens to be known. |
| Dangerous classification | Small marginal cost: retain and display another classification already present in the same Inspector result. | Low mechanical cost; potentially medium human review cost if every finding needs disposition. Default changes and added enum cases need contextual judgment. | Flags changes that can alter client behavior without invalidating documents. Neither composition nor operation validation replaces this. |
| Known-operation validation | Small to medium when given fixed GraphQL documents: parse sources and fragments, invoke validation, report affected documents and corpus identity. | Low to medium for the checker; medium or high for keeping an owned corpus representative across client versions. Collection, telemetry, retention, and freshness enforcement would be a separate, larger concern. | Identifies concrete affected supplied documents. Much structural evidence overlaps with schema diffing; useful attribution and executable-document failures remain. It cannot establish compatibility for absent clients. |
| Federation routing-change review | Medium for a conservative review surface over changed subgraphs, routing configuration, and composed metadata. High for a comprehensive semantic classifier or automated query-plan comparison. | Medium for human review and fixtures; high for evolving directive semantics, planner/configuration parity, plan noise, and runtime evidence. | Exposes changes hidden by the public API. Valid composition covers invalid routing relationships; it does not make every valid relationship operationally safe. |

The supporting library capabilities and limitations are detailed below. The
ratings are marginal where indicated: there are not five independent engines to
implement. Shared work includes immutable input identity, trusted tool/policy
selection, complete candidate assembly, a common result format, and handling
tool errors. Those workflow guarantees are required by the existing
[design requirements](../design.md#requirements-carried-forward), independently
of the number of checks selected.

## What the libraries provide

### Composition needs an exact engine and entry point

Apollo documents composition as producing both a combined schema and routing
metadata, and provides local composition through Rover. Its API schema omits
Federation machinery, making it the appropriate subject of client-facing
compatibility comparison. [Composition documentation](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/composition),
[schema types](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/schema-types).

In `apollo-federation` **2.19.0**, `composition::compose` expands and upgrades
subgraphs, validates them, merges them, and calls `validate_satisfiability`.
Its pre-merge stage contains a root-field consistency check and a TODO; its
post-merge stage returns success with a TODO for validations other than
satisfiability. `CompositionOptions` also says most JavaScript options have not
been ported. These are specific integration questions, not evidence that Rust
composition lacks all validation. [Versioned composition source](https://docs.rs/crate/apollo-federation/2.19.0/source/src/composition/mod.rs).

The separately exposed `apollo_federation::Supergraph::compose` calls the older
merge path and constructs a validated schema; it does not call the above
pipeline or its satisfiability stage. Selecting a method because it is named
`compose` would therefore be insufficient. [Versioned legacy entry point](https://github.com/apollographql/router/blob/v2.19.0/apollo-federation/src/lib.rs#L275-L289).

There are practical alternative integration paths. Apollo's JavaScript
`@apollo/composition` source exposes `composeServices`, validates subgraphs,
merges them, and defaults `runSatisfiability` to true. This is an existing
implementation that can fit the Node adapter, subject to the graph's required
Federation version. It is not a reason to assume every newer feature is covered.
[JavaScript composition source, moving branch inspected on the research date](https://github.com/apollographql/federation/blob/main/composition-js/src/compose.ts).

Apollo's current changelog says its Federation 2.15 composition release moved
to Rust. Language alone therefore does not distinguish a supported product
path from a partial library integration. The `apollo-federation` crate version,
Federation composition release, and router version must be treated as distinct
version identities. The changelog's product statement does not resolve the
library entry-point questions above. [Federation release documentation](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/versions).

Rover is a documented local alternative with a separately versioned composition
plugin. Apollo recommends an exact composition pin and warns that a router can
reject unsupported supergraph output, recommending a startup check with the
actual router. [Rover composition and versioning](https://www.apollographql.com/docs/rover/commands/supergraphs).

**Recommendation:** keep the CLI in Rust, but choose the composition integration
by demonstrated parity for the supported Federation features and router. Before
implementation commits to direct crate use, compare representative success and
failure fixtures with the selected supported Apollo composition release. The
TODOs alone neither prove a faulty composition result nor establish parity.
Do not reimplement missing checks merely to preserve a single-language binary.

### Breaking and dangerous changes are one diff

Inspector's diff returns breaking, dangerous, and non-breaking changes. Its
CLI fails on breaking changes by default; `dangerousBreaking` deliberately
changes that policy. The library's classifications and the registry's acceptance
rules are separate decisions. [Inspector diff behavior](https://the-guild.dev/graphql/inspector/docs/commands/diff).

The **8.0.1** source classifies a required input field added to an existing input
object without a default as breaking. Adding an input field with a default, or
changing an existing input-field default, is dangerous. Adding a value to an
existing enum is dangerous. These rules are already implemented; retaining their
output does not require a second analyzer. [Versioned input rules](https://github.com/graphql-hive/graphql-inspector/blob/%40graphql-inspector/core%408.0.1/packages/core/src/diff/changes/input.ts),
[versioned enum rules](https://github.com/graphql-hive/graphql-inspector/blob/%40graphql-inspector/core%408.0.1/packages/core/src/diff/changes/enum.ts).

For example, changing `limit: Int = 10` to `limit: Int = 100` leaves an operation
that omits `limit` valid while changing its behavior. This is a constructed
example derived from the default-change rule, not an executed test. A passing
operation check would not justify suppressing that finding.

**Recommendation:** retain all classifications, report structural breaking changes
as failed compatibility checks under the accepted
[repository-managed exception model](../adr/0001-repository-managed-compatibility-exceptions.md),
and present dangerous changes for explicit review. Whether dangerous findings block until acknowledged is an
unresolved workflow contract; adding the display does not itself settle it.

### Operation validation consumes a corpus; it does not create one

Inspector **8.0.1** accepts a schema and document sources, combines referenced
fragments, and invokes GraphQL document validation. It also supports deprecated
usage and optional document-complexity checks. Its programmatic defaults include
`strictDeprecated: true`, whereas the documented CLI flag defaults to false, so
an adapter must set intended options explicitly. [Versioned validation implementation](https://github.com/graphql-hive/graphql-inspector/blob/%40graphql-inspector/core%408.0.1/packages/core/src/validate/index.ts),
[CLI validation options](https://the-guild.dev/graphql/inspector/docs/commands/validate).

The inspected API accepts documents, not runtime variable payloads or service
responses. Passing it demonstrates validity of those supplied documents under
the selected schema and rules; it does not execute resolvers, test authorization,
or establish result equivalence. GraphQL separately specifies document
validation and request execution, including variable coercion.
[GraphQL validation](https://spec.graphql.org/September2025/#sec-Validation),
[variable coercion](https://spec.graphql.org/September2025/#sec-Coercing-Variable-Values).

**Recommendation:** accept an optional, explicitly identified local corpus in
the first release. Validate it against both baseline and candidate so existing
invalid documents are distinguishable from newly invalid ones. Report supplied,
missing, empty, and unreadable input distinctly; an explicitly supplied but
unreadable corpus is a check failure. Record document counts and identity, and
never use missing coverage to downgrade public API breaking changes. Corpus
collection or claims of production coverage require a separate ownership and
freshness decision.

## Public API unchanged, execution affected

These are static scenarios derived from documented semantics, not execution
results. In each case assume the public fields, types, and arguments remain
unchanged and the resulting federation still composes.

| Change | Effect that an API diff cannot express | Source |
| --- | --- | --- |
| Move `Product.inStock: Boolean!` from Products to Inventory using `@override` | The field's resolving service changes. A progressive override label can also change traffic allocation without changing its type. | [Override semantics](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/directives#override) |
| Change `shippingEstimate` from requiring `weight` to requiring `size weight`, with both fields already available | The router must obtain an additional prerequisite even when the client did not request it. | [Requires semantics](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/directives#requires) |
| Remove a path-specific `@provides(fields: "name")` while `name` remains resolvable elsewhere | The available route changes and may require another subgraph fetch. | [Provides semantics](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/directives#provides) |
| Change a subgraph's routing URL with identical SDL | The supergraph directs requests to a different endpoint; the public schema remains identical. | [Supergraph routing metadata](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/schema-types#supergraph-schema) |

Apollo also documents that valid schema/directive changes can increase planning
cost, and that plans depend on the supergraph, planner version, router
configuration, and directive usage. Identical operations need not keep the same
plan across those changes. [Query-planning limitations and inputs](https://www.apollographql.com/docs/graphos/routing/v3/query-planning/query-planning-best-practices).

**Recommendation:** initially provide a conservative review surface: list changed
subgraphs and routing configuration, retain the complete source diff, and show
the composed supergraph diff alongside the public API diff. Label this as
Federation change review, not automatic execution-safety classification. A
first implementation may flag more changes for review rather than attempt a
supposedly exhaustive directive allowlist. Structured summaries can follow
where they reduce demonstrated review noise; aliases, linked feature versions,
and unknown metadata must not silently disappear from review.

Generating before/after plans for supplied operations is a later, bounded form
of evidence. It adds planner/configuration parity and corpus maintenance, and
plan changes still need interpretation. Runtime equivalence, endpoint health,
resolver correctness, authorization, and deployment sequencing remain outside
these static checks. This boundary follows from the documented planning inputs
and the schema-only inputs inspected above; it is not a claim about tested
deployment behavior.

## Smallest coherent release recommendation

1. Compose the complete baseline and candidate with one explicitly pinned,
   verified engine configuration, including satisfiability; retain diagnostics
   and API/supergraph outputs.
2. Run one Inspector diff over the public API schemas and retain breaking,
   dangerous, and non-breaking findings. Keep the compatibility failure visible
   when repository controls allow an exception; the checker does not approve it.
3. Make routing configuration and Federation changes visible in the same review.
   Use the existing pull-request review surface; defer a separate approval
   service and an exhaustive semantic routing classifier.
4. Validate a caller-supplied operation corpus when present and state its limits.
   If mandatory known-operation assurance remains a release requirement, obtain
   an owned corpus rather than weakening the meaning of a pass.
5. Report exact inputs, tools, rules, corpus identity, and each check's status.
   Describe the result as static schema assurance. Approval and runtime safety
   remain separate states in the design.

This buys strong mechanical feedback at modest marginal cost after composition.
The largest discretionary expansion is attempting automatic routing-safety
judgment or owning operation telemetry. The best next product decision is
whether this bounded static assurance is an acceptable release contract, with
dangerous and routing changes left to explicit human review.

For a later implementation, independent fixtures should cover composition
failure and unsatisfiability, public field removal, changed defaults, a known
operation becoming invalid, an absent corpus, and identical API schemas with
changed routing metadata. Upgrade verification should exercise the chosen
engine and router combination. These are proposed verification cases, not
checks completed during this research.
