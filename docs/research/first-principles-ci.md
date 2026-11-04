# CI-first registry from first principles

2026-11-01. Independent design alternative, not an accepted amendment. Based on
the handoff, [current design](../design.md), and ADRs 0007–0014. GitHub capabilities
below were checked against current primary documentation; no hosted experiment
or implementation was performed.

## The strongest alternative

Use a portable graph checker and one central CI publisher. Keep source approval
in source PRs. Let maintainers merge control PRs using native GitHub review.
Make a retained, validated supergraph artifact the unit eligible for deployment.
Run no continuously hosted Selvedge process.

Distinguish three records:

- A **source commit** identifies a code revision; source controls determine its
  approval. It does not identify generated SDL or deployment.
- **Approved inputs** identify source-authorized SDL and maintainer-approved
  controls. Their combined configuration might fail composition.
- An **accepted snapshot** retains complete inputs, exact generated outputs,
  reports, and receipt. Only its artifact is eligible for explicit activation.

A desired-input branch can hold native maintainer merges and scoped automated
source updates. A publisher evaluates an exact desired revision and writes the
accepted snapshot separately. Invalid inputs never replace an accepted artifact.
Source schemas require no mirrored review PR. No environment model is needed.
Static checks do not establish deployed subgraph readiness; explicit gateway
selection and the external readiness responsibility remain unchanged.

This removes custom control-PR merging and the demand that every native input
merge atomically install generated output. It preserves coherent deployment
artifacts by moving the admission decision to artifact acceptance. The price is
real: approval no longer means a valid complete registry snapshot, and approved
inputs can accumulate behind failed composition. A later change might make
previously rejected work publishable without its caller retrying. Preventing that
requires explicit selection or terminal-intent bookkeeping; it is not free CI
behavior. This stages source updates instead of immediately accepting them
(ADR 0008), changes rejected-work behavior (ADR 0011), and replaces control
finalization (ADR 0013). It also replaces service authentication (ADR 0014).
ADR 0010's retained accepted snapshots and ADR 0012's source ordering can survive.
None of these contract changes is implicitly approved by this comparison.

## Caller interface and its concealed work

An illustrative common caller supplies one generated bundle:

```text
PR:        generate SDL → check(bundle, PR head) → validity + compatibility
postmerge: generate SDL → publish(bundle, stable intent) → receipt or failure
deploy:    select(receipt.registryRevision, artifactDigest)
```

The bundle includes subgraph, exact bytes, claimed build revision, generation
inputs, and stable intent identity. The adapter obtains source identity from
trusted GitHub context. Artifact references must resolve to an exact repository,
run/attempt, and artifact identity, never “latest successful build.” Authentication
and source approval do not prove code-to-SDL correspondence. Changed bytes for
identical declared generation inputs remain an integrity failure.
An uploaded or approved input is only staged; publication succeeds only after a
durable accepted receipt exists. The correspondence required between PR-checked
and postmerge SDL remains unresolved in both designs.

This small interface is credible only if the shared module owns collection,
identity verification, polling, scope checks, composition, receipts, and recovery.
Making every source workflow implement those steps fails the deletion test:
removing the hosted process has redistributed its implementation across callers.

**Source-PR trust:** a source-editable workflow calling a pinned checker is not an
enforced checker. GitHub Enterprise Cloud documents rulesets that select a
specific workflow, supporting PR and merge-group events. This could enforce a
trusted generator/checker arrangement without an external check-reporting App.
The organization's plan and actual settings are unknown. Separately bypassable
compatibility and mandatory validity require suitable separate enforcement;
they must not collapse into one bypassable result.
[Required workflows](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-workflows-to-pass-before-merging).

Generating SDL still executes source code. Run that without publication
credentials, then consume only bounded data in a fresh trusted checker job.
A standard generator convention simplifies onboarding but constrains existing
build pipelines; an artifact interface preserves them at the cost of run
association and waiting. Private registry reads, forks, cancelled builds, and
coordinated multi-repository bundles need actual integration evidence.

**Reporting:** requiring the shared GitHub Actions App and a check name does not
identify trusted workflow code. Native checks from dispatch workflows cannot
substitute for eligible PR checks; GitHub explicitly excludes such jobs from
required-check evaluation. Without required-workflow enforcement, a central
publisher must also become a trusted check reporter, including source-PR
association and cross-repository permissions.
[Expected producer](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-status-checks-to-pass-before-merging),
[Check eligibility](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks#checks-from-some-workflow-jobs-are-not-evaluated).

**Transport:** calling a central reusable workflow runs in the caller's context;
its `GITHUB_TOKEN` remains repository-scoped. It does not create a privileged
central publisher.
[Reusable context](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations#github-context),
[Token scope](https://docs.github.com/en/actions/concepts/security/github_token).
Cross-repository `workflow_dispatch` needs target Actions-write authority;
`repository_dispatch` needs Contents-write. The former also permits disabling
workflows. Distributing these credentials changes source authority and secret
management. Treating dispatch merely as a wake-up and independently retrieving
authorized source artifacts limits data authority, but does not remove that
availability authority. OIDC does not itself supply either GitHub REST credential.
[Workflow permissions](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event),
[Repository dispatch](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event).

Central polling avoids source-held dispatch credentials, but requires discovery,
cursor/replay rules, explicit publication markers, result delivery, and a policy
for expired inputs. Repeatedly scanning rejected inputs violates caller-owned
retry unless attempts are durably distinguished. Manually transferring artifact
IDs is another caller obligation, not an architectural simplification.

## Concurrency, retention, and errors

Actions now supports `concurrency.queue: max`, allowing 100 pending runs; overflow
is cancelled. Ordering is by when runs begin waiting, not source history. Default
concurrency replaces the pending run. This is useful bounded serialization, not
a durable source-order queue.
[Concurrency semantics](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency).

The publication module still needs these invariants:

- Bootstrap requires explicit authority; a missing ordinary baseline is an error.
- Source scope is checked against governing accepted controls; proposed controls
  cannot authorize themselves. Native approval of new controls is distinct from
  their effective adoption. Coordinated membership changes need combined authority.
- Preserve complete batches and unrelated updates. Source ancestry prevents an
  older delayed request restoring older SDL. A source merge queue cannot order
  independent repositories.
- Acceptance binds complete inputs, toolchain, outputs, and receipt. Any
  conditional-write race requires reconstruction and evaluation against the
  new accepted state, or a deliberately changed artifact-selection contract.
- Recover original accepted receipts before obsolete-source rejection; never
  replay an accepted request over newer state. Unknown write outcome is not
  confirmed failure. Validity rejection, integrity failure, supersession,
  unavailable dependencies, and contention exhaustion remain distinct outcomes.
- Compatibility failures remain visible; externally authorized source exceptions
  must not become unconditional publication failures.

Temporary Actions artifacts expire by default after 90 days. Use them for
transport, not accepted-state durability. Preserve accepted snapshots and receipts
in reachable Git history without automatic pruning; distribute the retained bytes
through immutable releases. A cancelled upload does not undo acceptance, and
retry must recover the original bytes rather than recompose.
[Artifact retention](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/download-workflow-artifacts),
[Immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/supply-chain-security#what-are-immutable-releases).

## A narrower hybrid with better locality

Keep the OIDC-authenticated publication module, but let Actions receive storage
commands. GitHub documents `issue_comment` against the default branch and requires
the workflow there; comments include both issues and PRs. A trusted workflow can
accept `/selvedge accept <exact-PR-head>`, authorize the actor, capture the approved
control change, and call the existing publication operation. It never executes
PR code. `workflow_dispatch` is another entry point but accepts a selected ref,
so the service must enforce the trusted workflow/ref, not just the event name.
[Event semantics](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows).

The workflow owns actor authorization under an explicit maintainer policy. The
service authenticates its narrow maintainer CI grant and applies scope, immutable
intent, control preconditions, and the existing acceptance lifecycle. OIDC claims
identify repository, event, ref, and workflow; actor authorization is a delegated
workflow decision, not a property of the request body.
[OIDC claims](https://docs.github.com/en/actions/reference/security/oidc#oidc-token-claims).

Actions owns run state and explicit reruns. Rejection terminates that run; the
service needs no webhook inbox, automatic replay, or rejected-work inventory.
Accepted receipts remain durable. Command lifetime, withdrawal, stale PR heads,
and rerun authorization must be defined: GitHub reruns retain the initial actor's
privileges even when another actor triggers the rerun.
[Actor semantics](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts#github-context).
This preserves caller-owned retry; it does not promise guaranteed command delivery.
It remains delegated command authority, not live native-review enforcement at
the final registry write. Choosing between those contracts remains unresolved.

## Dependency strategy and assessment

Keep composition, public-API comparison, operation validation, and report mapping
inside one graph-checker module. Pin the actual Apollo and GraphQL Inspector
integrations after representative-schema/gateway verification; no new GraphQL
algorithm follows from choosing CI. In-process dependencies need no adapters.
Filesystem/Git are local-substitutable dependencies tested with temporary
repositories. GitHub is a true external dependency behind a narrow internal
adapter with a test implementation; hosted enforcement needs separate experiments.
Avoid a generic CI-provider framework.

CI-first removes runtime deployment, health management, and HTTP intake, and
native maintainer merges avoid a custom review controller. It adds workflow
onboarding, private-data access, artifact handoff, credential distribution or
polling, asynchronous failure interpretation, and multi-record recovery. No source
repositories have been inspected to establish which cost dominates.

The strongest objection to the hosted design is rebuilding GitHub's job and
review machinery. The strongest objection to fully CI-first is hiding an equally
large publisher behind YAML while weakening submission isolation or retry
semantics. Artifact-first acceptance is worth choosing if approved-but-unusable
inputs and its retry consequences are acceptable. Under current retained-snapshot
and caller-retry contracts, prefer the narrow hybrid for storage intake, and
validate native required workflows plus transport on one real source repository
before claiming that removing the publication process simplifies the whole system.
