# Trusted checks and publication recovery

Research date: 2026-11-01. This supports the remaining
[design frontier](../design.md#current-decision-frontier). The authority and
artifact-store decisions are accepted; the mechanisms below are recommendations
and implementation constraints to verify. No host settings or workflows were
changed, and no publication or adversarial workflow experiments were executed.

Subsequent decision: [ADR 0010](../adr/0010-retain-validated-outputs-with-acceptance.md)
makes Git snapshots authoritative for accepted inputs and generated outputs.
The release recovery discussion below now applies to distribution of those
recorded bytes, not generation after registry acceptance.

The stable-intent recovery and shared control-finalization contracts were also
accepted in [ADR 0011](../adr/0011-caller-owned-retries-and-publication-receipts.md)
and [ADR 0013](../adr/0013-finalize-approved-controls-through-the-shared-publisher.md).
Mechanism recommendations below still require hosted verification.

## Trust the producer, not only the check name

An ordinary repository-local `pull_request` workflow comes from the synthetic
merge commit. A proposal that can alter that workflow can therefore alter the
invocation of an otherwise pinned checker. GitHub's event-security documentation
distinguishes this from the trusted workflow source used by `pull_request_target`.
[Event security comparison](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target).

GitHub rulesets can require a particular workflow. The ruleset API identifies it
by repository, path, and ref or SHA. Where available, a separately controlled
required workflow allows ordinary PR and merge-group events with a read-only
checker that treats the proposed configuration as data. Actual availability and
settings for the future registry host have not been established.
[Required workflows](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-workflows-to-pass-before-merging),
[ruleset API](https://docs.github.com/en/rest/orgs/rules#create-an-organization-repository-ruleset).

Required statuses can be restricted to an expected GitHub App, and GitHub warns
against duplicate job names across workflows. Inference: restricting a check to
the shared GitHub Actions App does not identify one particular trusted workflow.
The implementation must establish both trusted workflow execution and an
enforceable result identity.
[Expected status producer](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-status-checks-to-pass-before-merging),
[job-name ambiguity](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#about-branch-protection-rules).

On current GitHub.com, `pull_request_target` takes workflow code and its default
checkout/ref/SHA from the repository default branch, not an arbitrary PR target
branch. It has elevated-trust behavior and its availability depends on host event
policy. It can inspect proposal data with explicit least permissions, but must
not execute proposal-controlled code or configuration. It is not selected here
as a universal default or as proof of unspoofable check identity.
[Behavior change](https://github.blog/changelog/2025-11-07-actions-pull_request_target-and-environment-branch-protections-changes/),
[event security](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target).

The report must distinguish graph baseline, proposal head, combined candidate,
trusted workflow revision, checker identity, and governing policy. A workflow's
default `GITHUB_SHA` is not by itself evidence of which graph inputs were tested.
Explicit Check Runs API reporting selects a `head_sha` and requires
`checks: write`; any such reporter is a separate authority from read-only
validation. Verify actual PR/merge-group association in the host integration.
[Required-check evaluation](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks),
[Check Runs API](https://docs.github.com/en/rest/checks/runs#create-a-check-run).

The accepted compatibility-bypass contract also constrains workflow boundaries:
one indivisible required workflow failing for both validity and compatibility
cannot provide independent bypass of compatibility. Preserve distinct enforceable
requirements even if they share an analysis implementation.

## Proposed policy and tool changes

Recommendation: the accepted checker and policy govern both baseline and
candidate. Evaluate proposed controls separately; their proposal cannot replace
its own governing check. Separate control PRs simplify this, at the cost of an
extra step for a schema change that requires a new tool or policy.

Accept control changes through existing repository review. Evaluate existing
graph/corpus inputs under proposed controls and compare generated outputs, since
a tool upgrade may change artifacts without changing SDL. Exact checks and pins
remain to be established against the chosen composition integration. This is a
design recommendation, not an additional approval system.

## Publication inputs and bytes have different identities

Recommended input identity includes the source repository and accepted full
commit SHA, graph directory, exact comparison baseline, tool/composer identity,
effective policy/configuration, and baseline/proposed corpus identities. Record
the generated-byte digest separately and bind these identities in immutable
metadata. A source SHA alone cannot identify all outputs produced using different
tools or policies.

Recommendation: a tool change creates a different publication identity even
when SDL bytes match. Different bytes from the same declared inputs require a
visible reproducibility failure, not silent substitution. These are proposed
application contracts, not native API idempotency guarantees.

GitHub recommends creating a draft, uploading assets, and then publishing an
immutable release. Published assets and tag are protected, while release prose
and latest/pre-release metadata remain editable. The entire release can still
be deleted. Keep provenance in a protected asset and verify immutability for the
actual release, since enabling the feature applies only to future releases.
[Immutable release semantics](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases),
[enablement](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/establish-provenance-and-integrity/prevent-release-changes).

## Proposed retry and recovery contract

| Observed state | Required behavior under the proposal |
| --- | --- |
| Matching published immutable result | Verify source, inputs, and bytes; reuse it. |
| Matching incomplete draft | Resume missing uploads, then verify completion before publication. |
| Existing identity with conflicting source, inputs, or bytes | Fail visibly without replacement. |
| Uncertain response or timeout | Reconcile remote state before retrying or declaring success/absence. |
| Publication unavailable | Keep the source approved; do not expose an unverified new artifact for selection. Existing pinned gateway configuration is unaffected. |
| Older/newer publishers finish out of order | Preserve explicit identities. Completion order and a mutable latest marker do not activate a graph. |

GitHub's APIs require deliberate conflict handling: duplicate asset names return
422, and failed uploads can leave empty `starter` assets. A release creation
request's `target_commitish` is ignored if the tag already exists. Verify that
tag's resolved commit rather than treating the request as proof of source
identity. The release-by-tag endpoint concerns published releases; draft
reconciliation also needs publisher-visible draft state.
[Release assets API](https://docs.github.com/en/rest/releases/assets#upload-a-release-asset),
[Releases API](https://docs.github.com/en/rest/releases/releases).

A publish timeout may occur after remote success. Until reconciliation, the
truthful status is that publication outcome is unverified, not that no artifact
exists. This follows from the multi-request workflow; no atomic publication API
transaction is assumed. Same-identity publishers need coordination or conflict
handling that preserves these conditions.

GitHub's release attestations bind assets, tag, and commit; verification commands
check that integrity. They do not establish adequate schema validation, the
authorization of a compatibility bypass, the truth of submitted upstream source
claims, or deployed-subgraph readiness.
[Release verification](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/verify-release-integrity).
