# Implementation guidance and evidence

Status: planning record, 2026-11-04. This repository contains a design, not a
service, production SQL schema, CLI or deployment pipeline. This document turns
the [application contract](design.md), [data core](postgresql-data-core.md) and
[operations design](operations.md) into an implementation sequence and evidence
requirements. It does not authorize implementation or claim unrun checks passed.

## Decisions before implementation

The design's [open policy questions](design.md#unanswered-policy-questions) are the
authority for unresolved contracts. The following work makes them implementable;
record the chosen values beside their owning configuration or interface when it
exists, rather than maintaining a second set of pins and commands here.

| Boundary | Required decision or specification |
| --- | --- |
| Source publication | Resolve source ancestry versus CI release selection; exact grant/approval mapping; rejected-key and generation-consistency requirements. Do not ship a weaker policy by omission. |
| Request and receipt | Versioned request/response schemas; stable publisher identity across token renewal; exact normalization/equality rules; binary encoding; caller conditions; receipt-read access and historical format compatibility. |
| Checker and gateway | Pin composer, API extraction, diff/validation libraries, options and target gateway; verify licences and actual integration before selecting service/helper packaging. |
| Resources | Set input/corpus/artifact limits, checker time/memory limits, database timeouts and a shared contention/recovery budget; specify transport errors without claiming non-acceptance from timeout alone. |
| Database installation | Pin PostgreSQL and pgschema; specify graph bootstrap, role creation, effective owner selection, global default privileges and the plan/apply workflow. |
| Deployment | Confirm publication pauses and reliable old-writer exclusion; define bundle distribution, readiness and operational rollback checks. |
| Recovery | Define RPO/RTO, failover exclusion, backup/restore procedure and reconciliation of possible lost accepted history. |
| Empty-state policy | Resolve any required comparison to an earlier nonempty revision; otherwise immediate-baseline comparison is unavailable, not successful. |

The public API must distinguish stable caller conditions from the service's
changing evaluated baseline. A receipt identifies the graph, request key, accepted
revision, binding and exact evaluated result. Historical result access must remain
possible without submitting the original request to today's new-publication
validator. Specify which metadata each read grant exposes, rather than assuming
that artifact access includes all source provenance or receipt details.

## Implementation sequence

1. **Prove the checker and gateway boundary.** Use representative real Federation
   schemas and known operations with the proposed pinned tools. Compose, extract
   the public API, classify changes, validate operations and load the exact output
   in the selected gateway. Exercise composition failure and empty input. Compare
   language/packaging choices using those results; do not establish a Rust/Node
   process boundary before proving its necessity and cost.
2. **Build one vertical slice.** Start with a real PostgreSQL instance, two small
   subgraphs and the four relations. Implement authorized bootstrap, immutable
   reads, one complete publication module and the restricted SQL operation. Drive
   check → publish → receipt recovery → artifact read → bundled gateway startup.
   Use one source of truth for request binding and candidate inputs. Integrate
   pgschema and administrative bootstrap on this real schema from the outset.
3. **Prove contention and recovery at the application boundary.** Exercise the
   actual evaluation/retry loop, batch conditions, graph-wide edits, rejection
   freshness, outer rollback and response loss. A mocked revision resubmission
   alone cannot establish these guarantees. Add failure injection only where it
   exposes a meaningful application or transaction boundary.
4. **Integrate CI authority and reporting.** Implement the resolved grants, source
   ordering/approval association and Actions OIDC validation. Verify denial paths,
   fork/untrusted proposal handling, token renewal and report-target association.
   Source code generation executes in source CI; publication credentials stay out
   of proposal-controlled tools. Reporting failure must not undo or misreport an
   accepted registry revision.
5. **Prove operations before deployment.** Exercise exclusive writer handover,
   checker upgrade comparisons, schema upgrades/partial failures, restore gaps
   and gateway cold starts without the registry. Measure publication latency,
   contention and retained-data growth before declaring an operating envelope.

Keep the application and persistence boundaries cohesive. A small SQL function is
not a complete publisher: its caller still owns recovery, authorization, immutable
snapshot evaluation, retries and commit outcome. Conversely, transport adapters
must not duplicate those responsibilities. Add abstractions only when they make a
real operation or required evidence boundary clearer.

## Required behavioral evidence

These are acceptance scenarios derived from the design, not an implemented test
suite. Derive expectations independently from the contracts; compare complete
stored state and caller-visible outcomes. For a regression-detection claim, observe
the same expectations fail against the faulty behavior and pass after the fix.

| Scenario | Required observation |
| --- | --- |
| Initial empty/nonempty graph | Creation commits a real non-null current revision and complete inputs, or rolls back entirely. Ordinary publication cannot create a missing graph. |
| Real candidate evaluation | Exact candidate inputs produce retained outputs/report with pinned checker/options identity; local and service checks agree for the same complete inputs and scope. |
| Unrelated source updates | One attempt loses freshness, reloads and recomposes against the newer revision, and accepts only while its original participant conditions still hold. |
| Same participant, including equal SDL or A→B→A | Obsolete expected publication conflicts; byte equality cannot revive it. No batch participant is dropped. |
| Addition/removal and delayed add | Current absence permits addition; repeated accepted removal/addition recovers without changing state. A never-accepted delayed add can recreate a name after add/remove, as explicitly accepted. |
| Maintainer corpus/configuration edit | Any graph advance conflicts with its fixed expected registry revision; binding includes that condition. Source-only retries inherit the newer authorized maintained inputs. |
| Atomic batch and outer transaction | Stale participant, mid-function failure and rollback after function return leave no revision, publication, membership or current-pointer change. Acceptance is reported only after commit. |
| Accepted replay under later rules | Replay after intervening publications/removal or checker/request-validation changes returns the original authorized result without composition or rewind. Independent receipt read works with graph/key alone. |
| Concurrent duplicate and response loss | Concurrent identical attempts produce one acceptance; a lost commit response is recovered using the same binding. An in-flight commit after an absent read is never labelled definitively absent. |
| Negative outcome after uncertainty | Rejection, conflict, auth denial, resource error and timeout do not erase an earlier unknown outcome. Lost receipt-read access is not treated as non-acceptance. |
| Binding integrity | Changed bytes, participant conditions, graph-wide condition, source claims or publisher identity cannot reuse an accepted key. Legitimate token renewal does not change stable identity. |
| Rejection freshness | A baseline-dependent failure rereads the authoritative head; advancement causes retry or caller conflict. Repeated churn yields bounded operational failure, not a false composition verdict. |
| Compatibility/report fidelity | Baseline comes from retained schemas. Original corpus regressions remain visible despite corpus deletions. Approved compatibility exceptions retain failure classifications; absent corpus/schema evidence is unavailable. |
| Empty lifecycle | Empty → nonempty → empty preserves history and exact old bytes; absent artifacts remain null, empty revisions cannot start a gateway, and no implicit historical baseline is substituted. |
| Scoped authority | Every action and batch participant is authorized; source schema access cannot change routing/corpus/grants. Old controls authorize replacements. Reads do not reveal ungranted metadata. |
| OIDC/provenance/reporting | Invalid issuer/audience/signature/lifetime/claims are denied; report target and verified facts are bound correctly. Caller assertions and OIDC context are not misreported as generation or approval proof. |
| Database integrity and privileges | Cross-graph references and duplicate successors/initial revisions fail. Runtime direct writes, historical membership insertion, rewind, truncate and privilege escalation are denied. Exact bytes round-trip. |
| Schema rollout | Fresh install and upgrade converge with correct cyclic-FK order and effective privileges. Stale plans fail; partial apply is inspected/recovered without loss of accepted data or reopening incompatible writers. |
| Checker/control handover | Delayed old work cannot commit after the new release opens. Uncertain old transactions are reconciled. Upgrade evidence is bound to final quiescent graph/corpus revisions; a publication during draining triggers reassessment. Old evidence is preserved and tool-induced changes are identified. Stale reader instances/control caches cannot authorize reads after read revocation becomes effective. |
| Bundle startup and rollback | Correct bundle starts with registry unreachable; wrong/missing/corrupt/unsupported bundle stays unready. No different revision is substituted. Historical rollout checks actual subgraph compatibility. |
| Failover and restore gap | Old writer is excluded. Lost acknowledged receipts do not trigger automatic replay; affected writes remain suspended until the tested reconciliation procedure permits them. |

Source-ordering and rejected-key/generation-consistency scenarios must be completed
once those contracts are resolved. The accepted-only core cannot be described as
proving them. Include latency, retry counts, lock duration and retention growth in
load measurements; no numerical service target is specified yet.

## Observability and diagnostics

Expose the distinctions needed to operate these contracts: request key, graph,
evaluated and accepted revisions, checker/control release, retries, rejection scope
and whether commit outcome is known. Avoid dumping SDL, operation documents,
credentials or sensitive provenance into general logs; retained authorized records
are the source for detailed evidence. Preserve receipt identity through CI reporting
failures. A failed report upload is not a failed publication.

Measure evaluation time, contention/condition conflicts, recovery/unknown outcomes,
database lock/transaction failures, stored-byte growth and migration state. Gateway
deployments report the loaded artifact identity and readiness separately. These are
observability requirements, not a commitment to a metrics vendor or a telemetry
collection subsystem.

## Existing evidence

Isolated local PostgreSQL 18.6 fixtures exercised the four-relation model and
restricted function. Independent review assessed the design and experiment
results. The fixtures had no network access or published ports and used temporary
storage. Observations included:

- Concurrent requests against one evaluated revision accepted at most one new
  result; matching requests recovered one acceptance. Unchanged additions retried
  after unrelated advancement, while competing same-name additions conflicted.
- Changed request bindings and stale participant conditions failed. Accepted
  replay after later revisions returned the original result without rewinding
  current state; replay after removal did not recreate membership.
- Explicit rollback, stale batch participants and errors after inserting the
  first participant left no partial revision, publication or membership.
- Runtime history mutation, historical membership insertion, pointer rewind and
  truncation were denied. Cross-graph references, duplicate initial revisions
  and duplicate successors violated constraints.
- Binary outputs and UTF-8 SDL with CRLF survived exact byte round trips. Separate
  UUID/non-null/deferred-reference creation checks rejected incomplete graphs.
- Empty → nonempty → empty → recreated lifecycles preserved historical artifacts.
  Null artifacts serialized to JSON `null`. A never-accepted addition succeeded
  after intervening creation and removal, confirming the accepted absence rule.

The main fixture used integer revision labels and nullable bootstrap setup;
separate fixtures checked UUID and non-null creation. Request binding used a
fixture JSONB shape, not the final API. Composition outputs were fixture bytes;
recomposition was represented by resubmission against a new evaluated revision.
Actual response loss, HTTP, authentication, crash durability, failover and
backup/restore were not exercised. These results establish database-operation
evidence, not end-to-end or deployed guarantees.

Separate experiments with `@apollo/composition@2.14.4`, `@apollo/gateway@2.14.4`
and `graphql@16.14.2` rejected zero-subgraph composition with `NO_QUERIES`, empty
Gateway service lists and empty SDL. GraphQL requires a query root and at least
one field per object, so an empty `Query` is not a valid workaround. These results
support absent artifacts; they do not verify the eventual Rust integration or
the deployed gateway version.
[Apollo composition errors](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/errors),
[GraphQL root types](https://spec.graphql.org/September2025/#sec-Root-Operation-Types),
[object validation](https://spec.graphql.org/September2025/#sec-Objects.Type-Validation).

The pgschema review inspected v1.13.4 source and upstream fixtures for
[function security settings](https://github.com/pgplex/pgschema/blob/v1.13.4/testdata/diff/create_function/issue_335_search_path_rewrite/plan.sql)
and [null-aware uniqueness](https://github.com/pgplex/pgschema/blob/v1.13.4/testdata/diff/create_table/add_unique_constraint_nulls_not_distinct/plan.sql).
It did not execute the proposed schema through pgschema or establish installation,
upgrade, effective role selection, privilege convergence or partial-failure recovery.

The local probes are not shipped as an executable test suite. Documentation/link
checks and independent design review verify the record's consistency, not the
unimplemented system. Repository CI currently checks diffs only. Add setup and
validation commands when the corresponding pinned tooling and tests exist.
