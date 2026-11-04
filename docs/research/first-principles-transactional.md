# Alternative: one transactional registry

Status: unaccepted design alternative, 2026-11-01. This challenges implementation
choices in [the current design](../design.md); it does not amend its ADRs.

The essential operation is to accept one completely evaluated graph while
preserving its evidence. Git, release assets, Rust, and custom storage-PR merging
are mechanisms, not that requirement. A hosted registry with one transactional
durable store is a credible simpler design: accepted graph inputs, generated
bytes, reports, receipts, and the current head live together. Ordinary GitHub
review governs application and operator configuration. The registry never
merges PRs. Gateway deployments download an explicitly selected artifact.

The benefit is concentrated ownership and fewer storage lifecycles. It is not
stronger atomicity than the current self-contained Git commit already provides.

## Interface and invariants

One deep registry module exposes three operations:

```text
check(proposal, baseline = current) -> identified report
publish(intent) -> accepted receipt | identified failure
read(acceptedRevision) -> exact retained bundle
```

`publish` has separately authorized source, operator, and bootstrap variants;
three method names must not conceal a generic privileged patch operation.
Callers provide immutable schema bytes, source/generation claims, and a stable
intent identity. They need no storage handles, prepared candidates, or locks.

- Checks evaluate the complete federation, public API compatibility, optional
  known operations, and Federation review changes. Missing operation evidence
  remains unavailable. The baseline corpus stays fixed across comparisons;
  proposed deletions cannot erase regression evidence.
- Checks are read-only except for associated GitHub reports. Results identify
  exact candidate, baseline, tools, controls, and verified versus claimed facts.
  Source repositories own approval and compatibility exceptions. Publication
  requires validity but preserves failing compatibility results after an
  authorized source merge.
- Acceptance retains exact evaluated inputs and outputs indefinitely, without
  automatic pruning. Retries return the original receipt, even after later
  acceptances. Reusing an identity with different inputs is an integrity failure.
- Source history advances per subgraph, including unchanged SDL. Older or
  divergent publications cannot overwrite newer work. A coordinated batch is
  indivisible; missing ancestry evidence is an operational failure.
- A successful check is neither publication authority nor deployment readiness.
  Activation remains an explicit gateway deployment choice.

Failure results distinguish invalid composition, superseded source, conflicting
operator preconditions, denied authority, integrity failure, unavailable evidence
or tools, exhausted contention, and unknown acceptance. Unknown acceptance
requires the same recovery identity; it must never be reported as rejection.
Callers own explicit retry. There is no durable queue of rejected publications.

## Hidden work and the transaction

The implementation captures request bytes once, authenticates result access,
looks up any original receipt, and loads head `H`. Current grants then govern
new acceptance. It applies the authorized replacement to the complete graph,
verifies source ordering, and runs trusted, pinned analysis outside the database
transaction with bounded resources. Proposals cannot supply executable code;
any helper receives no publication credentials.

A short transaction checks that the head still equals `H`, then atomically
inserts the immutable snapshot and unique intent receipt and advances the head.
The snapshot contains its previous revision, complete consumed controls,
provenance, reports, and artifact bytes. Unrelated concurrent updates cause
bounded reconstruction and reevaluation under current controls. A failed
evaluation rereads the head before returning a semantic rejection. A lost commit
response is reconciled through the receipt before another acceptance attempt.

This requires atomic visibility, conditional updates, uniqueness, and the
selected durability guarantee, not a particular vendor. PostgreSQL documents
[atomic transaction visibility](https://www.postgresql.org/docs/current/tutorial-transactions.html)
as one feasibility example. Database choice and isolation behavior still require
verification; composition cannot become part of a long-held transaction.

Receipt lookup becomes an indexed key lookup rather than accepted-history
traversal. `read` serves stored bytes directly, with revision and artifact digests;
an interrupted download is retried. No release upload, second artifact store,
outbox, or distribution-success state is required initially.

## Operator changes and activation

Application code, tool pins, routing, ownership, and policy receive ordinary
protected GitHub PR review. After merge, trusted operator CI explicitly publishes
the exact control bundle, naming its predecessor control revision. Source
publishers cannot invoke this variant. Native repository controls determine
review authorization before CI submission; Selvedge does not implement reviewer
selection or reproduce merge decisions.

The registry evaluates the proposed controls against the latest graph, under
existing admission authority, including generated-output differences. Concurrent
schema changes trigger reevaluation. A different intervening control revision
returns a precondition conflict instead of silently overwriting it. Failed
publication leaves effective controls unchanged; the merged Git configuration
remains desired configuration requiring operator correction or explicit retry.

Deploying application code must not silently switch graph controls or tool
versions. A rollout supplies support for the effective and proposed toolchain
until explicit control acceptance succeeds. If the effective toolchain cannot
run, the registry reports unavailable analysis. Application rollback must remain
compatible with accepted data and effective controls; otherwise it requires a
forward repair. Root trust and emergency administration remain deployment
responsibilities, not powers a proposed graph grants itself.

Membership changes use an operator-authorized complete batch containing the
necessary schemas and controls. Proposed source grants cannot authorize their
own admission. Explicit bootstrap similarly requires setup authority and full
validity, with compatibility unavailable. These are proposed admission contracts,
not already settled behavior.

## Caller workflow

1. Products CI generates SDL and calls `check`; trusted results appear on its
   source PR against revision R17. Another subgraph can proceed independently.
2. Source controls permit merge, including any externally authorized
   compatibility exception. Trusted post-merge CI calls `publish` with one
   stable intent and its exact bytes. The registry evaluates against current
   R18 and accepts R19, or returns an identified failure.
3. If the response disappears, the same intent recovers R19 even after R20
   exists. It never reinstalls R19.
4. Gateway deployment downloads R19, verifies its digest, confirms external
   subgraph readiness, and bundles those bytes into its deployment. Existing
   gateways keep running during registry outages. Rollback still depends on
   deployed subgraphs supporting the selected older artifact.

## Dependencies and whole-lifecycle cost

Composition, compatibility analysis, and operation validation form a
graph-checker module. Its computation is in-process where possible; a pinned
helper, if required, is a local execution dependency. It also supports complete
directory inputs locally. Rust remains a preference; select the implementation
language after demonstrating engine and gateway fidelity.

The database is a local-substitutable dependency: test transactions against the
actual engine locally. GitHub authentication, check reporting, and source
ancestry are true external dependencies with narrow internal adapters and host
verification. OIDC provides repository/workflow claims for scoped trust, not
source-to-SDL proof; see [GitHub's claim reference](https://docs.github.com/en/actions/reference/security/oidc).
No speculative multi-provider or storage-adapter framework is needed.

Compared with the present direction, this removes Git snapshot construction,
accepted-history interpretation, hosted ref-write reconciliation, registry-write
credentials, release publication, and special storage-PR event/merge machinery.
It retains semantic retries, provenance, authority checks, source ordering, and
explicit configuration activation. A database does not resolve those concerns.

The added cost is a durable database deployment, access control, upgrades,
migrations, capacity monitoring, backup infrastructure, and restore drills.
Indefinite retention needs capacity planning for schemas, artifacts, and reports.
Use portable complete-bundle export; export is not a second authoritative store.
There is no application data to migrate today, but changing revision identity
and documentation has review and future integration costs.

Durability needs an agreed failure model and recovery objective. For example,
PostgreSQL can acknowledge transactions before durable flush under an
[asynchronous setting](https://www.postgresql.org/docs/current/runtime-config-wal.html);
calling a store transactional does not establish the required guarantee.
[Archive recovery](https://www.postgresql.org/docs/current/continuous-archiving.html)
depends on complete backup/log coverage and can deliberately restore an older
state. After restoration, publication stays disabled until accepted-history
completeness and current authorization are established. Missing receipts after
known data loss are not evidence that requests never succeeded. Irrecoverable
acknowledged data is reported as loss, requiring explicit resolution rather than
automatic rebootstrap. Backups must include artifacts and receipts, not only
current schemas; periodically prove exact-byte retrieval and replay recovery.

## Strongest objections and decision

The existing Git design already installs one coherent snapshot atomically.
Indexed receipt lookup alone does not justify a database. Ordinary operator CI
and fewer release mechanics could also simplify a Git implementation. Likewise,
three operations offer little advantage if callers must learn hidden config
activation and recovery rules through failures.

Prefer this alternative if the hosted registry remains required and reliable
database operation is already available: it gives ordinary application data one
owner and removes bespoke GitHub storage integration. Without that operational
capacity, the operational burden may exceed the application code saved. Workload,
artifact sizes, hosting constraints, and recovery objectives are unmeasured;
there is no evidence for selecting a vendor or claiming lower total cost yet.

## Exact decision changes and remaining evidence

Adoption would require explicit amendments:

- ADR 0005: retain complete-directory checker input; remove registry-repository
  topology as canonical storage.
- ADR 0010: replace Git commits/releases with transactionally retained snapshots
  and direct exact-byte retrieval; preserve retention and explicit activation.
- ADRs 0008–0009: replace Git recording/conditional commits with store acceptance;
  preserve source approval, snapshot checks, and internal contention handling.
- ADR 0013: normal merged operator configuration plus explicit CI publication,
  predecessor-control preconditions, and effective-control activation.
- ADR 0014: retain scoped OIDC and GitHub reporting; remove registry-write
  credentials. ADR 0006 stays superseded. ADRs 0001–0004, 0007, 0011–0012 retain
  their guarantees. Q27 custom storage-PR merging is unnecessary.

The glossary's Git-specific registry revision becomes an immutable accepted
revision; source Git revisions retain their meaning. Checked-versus-published
SDL correspondence and legitimate same-source regeneration still need explicit
contracts. Verify real schemas, gateway consumption, concurrent publication,
lost responses, stale controls, unauthorized inputs, and restore/replay through
these interfaces. This is static design analysis, not execution evidence.
