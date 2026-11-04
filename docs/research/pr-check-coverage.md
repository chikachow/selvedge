# Source PR coverage and check freshness

Research date: 2026-11-01. This answers how to detect stale or interacting schema
changes while source PRs remain open, including missing event handling. The
mechanisms below are recommendations, not an implemented integration or an
accepted cross-repository merge guarantee. No live repositories or event flows
were exercised.

## Discover work before publication

A publication handler cannot establish that every source PR was checked.
GitHub Apps have one webhook receiving configured events for repositories the
installation can access; a separate handler does not need to be installed for
each PR. Access permissions, event subscriptions, and the registered repository
inventory still define the scope.
[App webhook scope](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps#webhooks).

GitHub documents separate PR, push, workflow-run, and check-run events. A
workflow-run event cannot reveal a schema-generation workflow that never ran.
An unrelated registry push does not automatically rerun a source PR check:
GitHub does not know that the registry was an input to that earlier result.
The latter is an inference from event and commit-check boundaries.
[Webhook events](https://docs.github.com/en/webhooks/webhook-events-and-payloads),
[workflow events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request),
[Checks API](https://docs.github.com/en/rest/checks/runs#create-a-check-run).

Recommended coverage:

| Trigger or discovery | Service action |
| --- | --- |
| Source PR opens, changes head, reopens, or changes target | Ensure generation and a check exist for the current source inputs |
| Source target branch advances | Revisit candidates built from a test merge with that branch; old SDL may need regeneration |
| Registry graph, policy, operation corpus, or checker changes | Invalidate affected evidence and recheck open candidates for that graph |
| A generation/validation result arrives | Associate it with exact source and check inputs; a superseded result cannot satisfy the current check |
| Service starts or periodic reconciliation runs | Enumerate open PRs in configured source repositories and discover missing or stale records, including PRs never seen before |
| Source PR closes | Remove it from the active-check set; preserve any independently accepted publication work until resolved |

GitHub does not automatically redeliver failed webhooks. Periodic reconciliation
against open-PR and check APIs therefore repairs gaps that an event-only system
would retain. The reconciliation recommendation is an engineering inference;
its frequency and storage are not settled.
[Delivery failures](https://docs.github.com/en/webhooks/using-webhooks/handling-failed-webhook-deliveries),
[open PR inventory](https://docs.github.com/en/rest/pulls/pulls#list-pull-requests),
[checks for a ref](https://docs.github.com/en/rest/checks/runs#list-check-runs-for-a-git-reference).

Each source repository still needs a known way to generate Federation SDL.
Generic PR metadata cannot infer it from arbitrary code. Missing generation
evidence is different from verified unchanged SDL; neither path filters nor the
absence of a submission proves that a PR has no schema effect. Reconciliation
can request a configured generator or report missing evidence, but cannot invent
the generation process.

## Retain enough information to repeat a check

Recommended active-check identity and inputs:

- Source repository and PR, current head, target/build revision, and affected
  subgraphs.
- Candidate SDL digest and retrievable exact bytes, or a regeneration path.
- Checked registry revision, trusted checker/policy/configuration, and operation
  corpus identities.
- Desired check generation and the result associated with that generation.

This is lifecycle-scoped source-PR work state, not another registry review
object. Old heads can be superseded and closed PRs removed from the active
inventory. Candidate payload retention must be bounded, and expired inputs must
produce regeneration-required evidence rather than leaving an unsupported pass.
Do not discard an accepted asynchronous publication job merely because its
source PR is now closed.

## Staleness is not incompatibility

A changed input identity invalidates the old evidence. It does not prove that
the schema is incompatible. To determine the current result:

1. Read the latest accepted registry snapshot.
2. Overlay only the candidate's permitted subgraph SDL, preserving current
   maintained configuration and other subgraphs.
3. Compose the complete baseline and candidate.
4. Compare the current baseline API with the candidate API; validate the fixed
   current baseline corpus against both; regenerate Federation notices.
5. Publish the result only if its source and governing input generation are
   still current. Preserve older results as historical evidence.

Start with graph-wide invalidation and complete checks. Reuse baseline
composition and coalesce superseded work as optimizations; do not rely on an
unproven dependency filter to decide that a changed subgraph cannot affect
another proposal. An update to the same subgraph also raises a source-ordering
question: successful composition alone does not authorize overwriting newer
source state with an older candidate.

## Required checks and their limits

Missing checks block merging only when the source repository requires them.
GitHub evaluates the relevant latest commit or tested merge revision; earlier
commit success is insufficient. Workflow filtering can leave required checks
pending, whereas a skipped job may report success. Source repository rules
must require the service's expected App identity and preserve the agreed
external compatibility-bypass policy.
[Required checks](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks),
[branch protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#require-status-checks-before-merging).

Invalidation must make an outdated green cease satisfying the effective check.
The Checks API supports updating a run and creating another run, but Apps cannot
set the literal conclusion `stale` or literal status `pending`. Rerequesting
resets the suite without updating the run itself. The exact update/replacement
sequence must be tested against branch protection; a rerequest alone is not
evidence that the earlier pass has been withdrawn.
[Check update and rerequest semantics](https://docs.github.com/en/rest/checks/runs#update-a-check-run).

Events and periodic repair provide eventual coverage and freshness. They do not
make registry changes atomic with source merges: a source can merge during the
window before its old green is invalidated. Conditional publication remains the
registry-write guard, separately from externally owned approval policy.

## Pending proposals can also conflict before registry state changes

The two-source-PR field-type example in
[the workflow research](source-pr-workflow.md#source-merge-and-registry-acceptance-remain-different-guarantees)
can fail in combination even when neither proposal has been published. Tracking
both does not itself check their combination.

Additional active-proposal comparisons can reveal such conflicts early. They
must be labelled as additional evidence: alternate PRs may not be intended to
merge together, multiple proposals can target the same subgraph, and pairwise
success does not establish safety of every larger combination. Checking every
possible subset grows exponentially. An intentionally coordinated change should
identify the complete candidate and participating source revisions explicitly.

The material guarantee to choose is whether the system must prevent incompatible
source PRs from both merging, or provide early checks/rechecks while preventing
an unchecked combined registry update. The stronger source-merge guarantee
requires explicit cross-repository coordination beyond asynchronous rechecking.

## Revised request-driven alternative

The user accepted snapshot-scoped feedback and conditional registry writes,
then challenged the complexity of automatic event-based revalidation and the
webhook integration it would introduce. The accepted concurrency boundary is
recorded in [ADR 0009](../adr/0009-snapshot-checks-and-conditional-registry-updates.md).
Automatic coverage and refresh are being reconsidered; the event/reconciliation
mechanism above is not an implementation commitment.

Recommendation: source CI explicitly calls a synchronous check API and owns
retries; an authorized post-merge caller requests subgraph publication against
identified inputs and expected registry state. The service does not maintain an
active-PR inventory or proactively refresh earlier results. Missing generation
or check results remain the responsibility of configured source workflows and
repository rules. Conditional writes still prevent a registry mutation from
using a snapshot that changed during its validation.

This can use a GitHub App without webhook subscriptions. GitHub explicitly
permits disabling App webhooks without providing a webhook URL, while
installation tokens can make outbound check and repository-write API calls.
Source Actions can trigger on PR activity and invoke the service without the
service receiving those GitHub events itself.
[Optional App webhooks](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/using-webhooks-with-github-apps),
[Checks API](https://docs.github.com/en/rest/checks/runs#create-a-check-run),
[repository-write API](https://docs.github.com/en/rest/repos/contents#create-or-update-file-contents),
[source workflow triggers](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request).

The cost is explicit: a successful source check is evidence about its recorded
registry and candidate, not a continuously refreshed claim about current state.
Independent proposals can become incompatible later; publication conflicts
return to the caller's workflow. No automatic freshness, PR discovery, or event
recovery guarantee is made by this alternative.

If proactive refresh is retained, polling can replace webhooks, but it still
needs candidate availability, lifecycle handling, scheduling, and recovery.
That is a different choice from making the service request-driven. The simpler
request-driven boundary is the recommendation awaiting confirmation.
