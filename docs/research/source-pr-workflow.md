# Source PR checking and a proposal service

Research date: 2026-11-01. The requirement to assess schemas before source PRs
merge is accepted; the service responsibility split and source/registry merge
ordering remain proposals. No concrete gateway or subgraph repositories are
available yet. This is documentation research and static analysis, not an
implemented or exercised integration.

## Check the code proposal's generated schema

Apollo documents source-CI schema checks with results shown on the source PR.
Rover also supports composing a registered graph with a local subgraph override
before publication. These support the basic workflow: generate the proposed SDL,
materialize an identified registry snapshot, replace the permitted subgraph input,
and check the complete federation. Selvedge retains its agreed structural public
API comparison rather than relying only on historical operation usage.
[Source-PR integration](https://www.apollographql.com/docs/graphos/platform/schema-management/checks/github-integration),
[schema checks](https://www.apollographql.com/docs/graphos/platform/schema-management/checks),
[local composition](https://www.apollographql.com/docs/rover/commands/supergraphs).

The report should bind source repository, PR head, actual build revision, SDL
digest, registry baseline, complete candidate identity, and trusted tools/policy.
GitHub's ordinary PR checkout normally uses a synthetic test-merge revision;
the displayed PR head is a different identity. Squash/rebase merging can produce
another revision. Recommendation: regenerate after source merge and confirm
schema identity before attributing the final revision to the registry proposal;
changed SDL needs renewed validation.
[PR revision semantics](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request),
[merge methods](https://docs.github.com/en/pull-requests/reference/pull-request-merges).

The source check's public API delta is relative to its stated registry baseline.
If that baseline already differs from the source target branch's generated SDL,
the report must not claim that every difference was introduced exclusively by
this source PR. No deployment state is inferred from either repository's head.

## Earlier coordinator proposal

The following option automatically mirrored source proposals into registry PRs.
The user subsequently raised the orphaned-PR risk; the alternative at the end of
this note and the current design discussion revisit that lifecycle.

| Component | Responsibility |
| --- | --- |
| Subgraph CI | Build the identified candidate, export Federation SDL, submit it with source identity |
| Proposal service | Authenticate submissions; enforce permitted subgraph scope; manage linked registry proposals; dispatch validation; report authoritative source-PR checks |
| Trusted validation CI | Run the pinned CLI against explicit complete baseline and candidate snapshots |
| Registry repository controls | Review, accept, or reject configuration changes and authorize compatibility exceptions |
| Post-registry-merge CI | Generate and publish the agreed immutable artifacts |
| Gateway deployment | Select the approved artifact explicitly |

Recommendation: make the service a GitHub App coordinator, while retaining the
existing CLI/CI validation path. This avoids introducing another execution path
for composition and keeps graph-domain logic independent of GitHub. It would
bring automated registry proposal creation into the first release, revisiting
the earlier deferral. A hosted composer remains an alternative if actual latency,
cost, or execution requirements justify it; none have been measured yet.

## GitHub provides the required primitives

A GitHub App installation token with Checks write permission can create checks
on a specific `head_sha`. Repository protection can require that App as the
producer. This gives the coordinator an identity distinct from arbitrary
source-repository workflows using the shared GitHub Actions App.
[Checks API](https://docs.github.com/en/rest/checks/runs#create-a-check-run),
[expected check producer](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#require-status-checks-before-merging).

The service can dispatch a trusted workflow using Actions write permission.
The documented dispatch ref is a branch or tag name, so choose and protect that
ref and verify the actual workflow commit; pass immutable graph baseline and
candidate SHAs separately. The workflow must exist on the default branch. Pin
the API version and establish dispatch/run correlation for that version.
[Workflow dispatch API](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event),
[event semantics](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#workflow_dispatch).

Current GitHub documentation says job checks from `workflow_dispatch` do not
satisfy PR ruleset requirements; external App checks are exempt. Therefore the
coordinator's verified result reporting is necessary for this proposal, rather
than assuming the dispatched job's native check is the source PR's gate.
[Required-check evaluation](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks#checks-from-some-workflow-jobs-are-not-evaluated).

Actions OIDC provides repository/run/workflow identities and a configurable
audience. Alternatively, a verified webhook can prompt retrieval of a specific
run artifact, whose metadata includes run, repository, head SHA, and digest.
These authenticate submitting context or artifact identity; they do not establish
that the SDL accurately describes a deployed binary. The exact submission
authentication contract remains to be selected.
[OIDC claims](https://docs.github.com/en/actions/reference/security/oidc),
[artifact metadata](https://docs.github.com/en/rest/actions/artifacts#get-an-artifact),
[webhook verification](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries).

## Persistence and recovery are separate from hosting

Git branches/PRs can retain submitted SDL and provenance. That does not itself
guarantee delivery of validation or reporting work. GitHub does not automatically
redeliver failed webhooks, and deliveries can arrive out of order.
[Failed deliveries](https://docs.github.com/en/webhooks/using-webhooks/handling-failed-webhook-deliveries),
[delivery ordering](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/troubleshooting-webhooks#webhooks-deliveries-are-out-of-order).

Recommendation, conditional on accepting the coordinator: persist proposals
before acknowledging acceptance, and reconcile unfinished validation/reporting.
The same source PR may have several revisions; an old result must stay attached
to its checked identity and cannot stand in for a newer candidate. A separate
application database is not assumed, but a recovery mechanism must be specified
before claiming that accepted proposals survive interruption.

## Source merge and registry acceptance remain different guarantees

| Source merge condition | Benefit | Consequence |
| --- | --- | --- |
| Required check against a recorded registry snapshot; registry acceptance follows source merge | Early developer feedback without requiring registry acceptance of unmerged source code | Concurrent changes can cause a later registry check to reject already merged source code |
| Registry acceptance of the exact generated schema before source merge | Central acceptance is established before the code lands | Requires linked identities and rules for changed/abandoned source PRs; approved registry state can describe code that never merges |

Neither option makes the two Git repositories transactional. A source merge
queue coordinates its own target branch, not all other repositories.
[Merge-group behavior](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#merge_group).

Static concurrency example: two source PRs independently add `Query.status`, one
as String and another as Int, in different subgraphs. Each can compose with the
same original baseline, while their combination has a field type mismatch.
Final registry acceptance must check the combined configuration against its
then-current target. The example is reasoning from the documented composition
error, not an executed fixture.
[Federation error reference](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/errors).

The user must choose the required source-merge guarantee. A service can implement
either policy; its existence alone does not select an approval order or prove
safe subgraph deployment.

## Avoid duplicating the source PR's review lifecycle

Automatically creating a registry PR for each source PR creates another object
that can outlive an abandoned, superseded, or substantially changed source
proposal. Closing stale PRs automatically would manage that lifecycle rather
than establish that the second review object is necessary. This is a static
failure analysis, not an observed production incident.

Revised recommendation for discussion: use the source PR as the proposal. Its
check requests generate results without creating registry branches or PRs. After
source merge, an explicit publication request identifies the merged revision
and SDL. The service verifies authority and allowed subgraph scope, checks the
complete candidate against the current registry, and conditionally records the
owned-subgraph update. Artifact generation and activation stay separate.

This delegates routine schema acceptance to protected source branches and gives
the service registry-write authority; it changes the earlier central registry
approval boundary. Registry maintainers retain policy/routing/ownership control.
Repository protections must deliberately permit the service's bounded direct
updates; a write-capable token alone does not establish that authority.
[Ruleset App bypass modes](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#granting-bypass-permissions-for-your-branch-or-tag-ruleset).

GitHub's `createCommitOnBranch` accepts a required `expectedHeadOid`; the service
can require the branch to remain at the revision it validated. GitHub documents
GitHub App use of this mutation. The exact token permissions and protection
behavior still require an integration experiment.
[Conditional commit API](https://docs.github.com/en/graphql/reference/commits#createcommitonbranchinput),
[GitHub App commit support](https://github.blog/changelog/2021-09-13-a-simpler-api-for-authoring-commits/).

The proposed workflow retains several explicit limits:

- A later registry state can reject a schema that passed before source merge.
  The source publication job must surface that failure and own its recovery.
- Conditional registry writes do not prevent a delayed older source request
  from replacing a newer publication of the same subgraph. Source ordering and
  intentional rollback need their own contract.
- A new compatibility break against a later baseline is not silently authorized
  by the earlier source review. Deliberately requested registry PRs can provide
  a native repository exception route; the exact policy is still open.
- Coordinated multi-subgraph changes must remain possible as a complete batch,
  without assuming independent publications always compose.
- Request persistence and completion still need guarantees. Eliminating
  automatically duplicated PRs is not proof of an orphan-free service.

If central human approval of every schema were retained, registry PRs could be
created only on explicit publication requests after source merge. That would
reduce abandoned candidate PRs while retaining a second review lifecycle.

Subsequent decision: source PRs own schema approval, including exceptions through
externally managed source-repository controls. The separate registry exception
route above was not adopted. See the authoritative
[source approval decision](../adr/0008-source-prs-own-schema-approval.md) and
[exception boundary](../adr/0001-repository-managed-compatibility-exceptions.md).
The service must distinguish request freshness from approval policy without
implementing another exception-approval mechanism.
