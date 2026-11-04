# A narrow Git-backed publisher from first principles

Proposal, 2026-11-01; no accepted decision changes. This reconsiders the
[current design](../design.md), including ADRs 0010 and 0013, against the original
handoff. Q27 remains open. Primary documentation was checked; no hosted behavior
or implementation was tested.

The strongest Git-backed design is one request-driven publication module owning
one machine-written accepted history. Git is the durable record of complete
accepted graphs, rather than the place where humans must perform every kind of
approval. Source PRs remain the schema approval surface. Maintainer changes use
an explicitly authorized, immutable request. Software upgrades use the ordinary
application release process. These are proposed authority choices, not facts
established by choosing a branch layout.

## Interface and ordinary use

| Operation | Caller responsibility | Module guarantee |
| --- | --- | --- |
| `check(proposal)` | Submit identified SDL and source/build context | Complete-federation analysis against a named accepted revision; trusted source-PR reporting |
| `publish(intent)` | Supply stable identity and authorized source replacements | Compose against current accepted state, enforce source ordering, recover original acceptance |
| `apply(expectedRevision, change)` | Authorize exact maintainer inputs and their predecessor | Accept the complete change and outputs together, or change nothing |
| `readAccepted(revision)` | Select an accepted revision | Return its retained bytes and evidence without recomposition |

These are complete operations, not necessarily four network endpoints. A local
checker accepts complete directories; callers never assemble Git transactions.

Source CI generates SDL, requests a check, and waits. Source repository controls
decide merge and compatibility exceptions. Trusted post-merge CI publishes.
Checks identify PR head, build revision, SDL bytes, registry baseline, and checker
release. Checking cannot publish. Publication still requires validity, while
retaining compatibility failures instead of introducing a second exception gate.
OIDC authenticates job context; it does not establish source-to-SDL correspondence.

For a routing or corpus change, an operator previews an immutable change against
revision H and submits `apply(H, change)` through trusted maintainer CI. The
initial rule rejects a changed H, even for independent schema advancement. This
deliberately trades occasional re-preview for eliminating control rebasing,
ordering, and stale-assumption rules. Source publication retains internal retries.
If operator contention proves material, introduce a narrower predecessor over
maintainer-owned inputs with explicit semantics.

Gateway deployment selects `readAccepted(revision)` and bundles those bytes.
Approval, observed subgraphs, and gateway activation remain distinct. GitHub
releases can be optional distribution copies; removing their mandatory lifecycle
would explicitly revise ADR 0010, while preserving exact retrieval and retention.

## Which state belongs where

The accepted tree contains schemas and source progress, routing and membership,
known-operation documents, graph-specific publication grants, generated
supergraph/public API bytes, reports, and the acceptance receipt. All changes to
these inputs pass through the same finalizer. Baseline operations remain present
in comparison evidence even when a proposed corpus removes them.

Checker code, dependency versions, and fixed classification policy belong to an
immutable application release. Snapshots record that release identity and its
consumed configuration; they need not contain editable instructions selecting
another executable. Authentication bootstrap, secrets, and deployment settings
remain operator configuration. Source grants and graph ownership stay with the
accepted graph, preserving current-control admission and revocation on retries.

This replaces ADR 0013's tool/policy PR finalization with reviewed application
release admission. It does **not** preserve the stronger promise that the old
checker governs every tool/policy transition. That promise requires a separate
upgrade protocol and should be retained if actually required. Application review
must cover policy changes and tool-induced output differences; changing their
storage location does not make them harmless.

There is no second desired graph in deployment configuration, no control revision
awaiting application, and no automatic startup recomposition. An upgrade leaves
previous artifacts intact; subsequent reports identify both the baseline and new
release, including tool-induced differences. An explicit unchanged-input apply
can record a newly generated artifact. Initially, upgrades quiesce the writer and
resolve outstanding acceptance attempts before switching releases. Mixed-version
writers or uninterrupted upgrades require additional fencing; they are not free.

Routing, corpus, and membership updates must therefore remain graph operations.
Coordinated membership changes require one combined intent with explicit
authority for maintained inputs and every source participant. Ordinary source or
maintainer grants must not silently acquire the other's permissions. The concrete
combined grant remains a contract to settle.

## Maintainer admission alternatives

**Head-pinned admin apply is the smallest interface.** A trusted manual workflow
accepts an immutable change identity and expected accepted revision. An authorized
maintainer's invocation delegates one bounded attempt; later cancellation cannot
promise to revoke an already accepted write. No storage PR is required. This is
an explicit replacement for mandatory maintainer PR approval in ADR 0013, so it
requires agreement. A PR may still provide review evidence without becoming a
second accepted history.

**A comment command can be a thin caller.** A trusted GitHub Actions workflow can
handle `issue_comment`, validate actor authority, bind the command to the exact PR
head, and call the same service with OIDC. GitHub documents that this event uses
the default branch and covers issue and PR comments; the workflow must distinguish
them. [Workflow events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#issue_comment).
Fetch proposal files as data; never execute their workflows. The service verifies
the trusted workflow grant and current graph scope. PR-head identity binds the
proposal; expected registry identity protects its predecessor. Neither replaces
the other.

GitHub's workflow run owns the attempt and failure result; explicit reruns can
retry. The service needs no webhook inbox or rejected-request ledger. Lost
accepted responses still recover through receipts. This is simpler than having
the service receive webhooks, acknowledge within GitHub's ten-second window, and
own intake recovery. [Webhook timing](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks#respond-within-10-seconds).
Commands nevertheless add parsing, actor/retry rules, PR completion reporting,
and a delegation policy. They do not establish native review enforcement at CAS.

**Native PR completion preserves a different guarantee.** GitHub's merge input
pins the PR head but exposes no expected accepted-base OID or supplied result
tree. [Merge parameters](https://docs.github.com/en/rest/pulls/pulls#merge-a-pull-request).
With direct source publications advancing that base, matching retained outputs
to the final merge needs additional write coordination or a verified fresh
candidate workflow. Updating generated files can invalidate approvals under
configured review-freshness rules. [Protected-branch reviews](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#require-pull-request-reviews-before-merging).
Use this option if native completion is a required admission guarantee; a custom
write plus a later merged badge is not equivalent.

## Hidden implementation and irreducible guarantees

The finalizer captures immutable request bytes and verified identity, checks
scope, then searches accepted receipts before rejecting an old source revision.
It loads H, applies only authorized replacements, checks every participant's
source ancestry, composes, compares, and creates one child commit containing
inputs, outputs, and receipt. It advances the accepted ref only if it still
equals H. GitHub's `updateRefs` documents `beforeOid` for this condition.
[Conditional reference update](https://docs.github.com/en/graphql/reference/git#updaterefs).

Source contention repeats scope, ordering, and evaluation against the new head;
rejection also rechecks head freshness. Retries are bounded. An uncertain write
is reconciled before replay; uncertainty is never reported as proven absence.
Maintainer predecessor failure instead requires an explicit refreshed request.
No rejected publication becomes future background work, and no coordinated batch
silently loses an overtaken participant.

Callers distinguish accepted/recovered, invalid, superseded/conflicting,
unauthorized/integrity failure, unavailable evidence/infrastructure, and unknown
acceptance. An absent ordinary baseline is an error; first creation is an
explicitly authorized bootstrap with unavailable compatibility evidence.

Four identities remain necessary: publication intent, source progress, accepted
commit, and generated-byte digest. Reports also identify directional comparison
inputs and tool release. Equal SDL cannot prove source ordering, request identity
cannot identify bytes without binding, and a digest cannot promise retention.
Receipts live in the linear accepted history; indexes are rebuildable conveniences.
Retained history has no automatic expiry, with backups and protected refs needed
for the promised durability. Only the publisher writes accepted snapshots.

## Depth, dependencies, and strongest objections

The publication module earns its depth by hiding assembly, Git concurrency,
authorization enforcement, lineage, and recovery from every caller. The checker
owns composition and diagnostics. A narrow GitHub adapter owns OIDC, source
evidence, reporting, and Git transport. Prefer one Rust executable with the
established Apollo/Inspector integrations; exact versions and gateway behavior
still need validation. Local computation uses real fixtures, Git uses temporary
repositories, and true external GitHub dependencies require mock adapters plus
hosted contract tests. Do not expose those internal seams to publication callers.

The remaining cost is substantial: a privileged hosted process, trusted source
workflows, release operations, retained Git growth, lineage access, and uncertain
network-write recovery. Git saves a database; it does not remove application
transactions. Linear receipt scans and strict admin preconditions need measurement.
Service downtime blocks publication and trusted checks. No availability target is
known, so worker fleets, caches, and queues are premature.

The strongest objection is authority: explicit admin invocation and ordinary
software review may be unacceptable substitutes for mandatory control PRs and
old-policy admission. Retaining those contracts favors the Actions-command caller
with controls still in accepted snapshots, sacrificing some simplification.
If callers already operate trusted CI, the hosted process itself must also justify
its operational burden against a CI-only implementation.

The unresolved checked-versus-published SDL relationship and generation identity
remain unresolved under every storage choice. Fresh composition cannot establish
earlier review of different bytes. Before implementation, settle those contracts
and maintainer delegation, then test complete composition, baseline-corpus
retention, concurrent publication, lost responses, stale control inputs, scoped
authority, and exact historical retrieval through the public operations.
