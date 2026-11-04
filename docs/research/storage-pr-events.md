# Selvedge-owned storage PR finalization

Research date: 2026-11-01. Current GitHub primary documentation was read; no
hosted writes, branch-rule experiments, or integration tests were performed.
This revisits the pending control-review workflow in
[the design](../design.md#current-decision-frontier). It does not accept a new
architecture or supersede an ADR. The earlier
[control PR note](control-pr-finalization.md) retains the detailed merge/ref
comparison.

**Assessment:** a GitHub App can receive storage-repository PR activity and
comment commands, and repository rules can restrict accepted-branch updates to
that App. This makes one accepted branch feasible without the separate
reviewed-control history. The material choice is whether GitHub's native merge
enforces current review requirements, or Selvedge receives authority to finalize
an immutable control proposal itself. Webhook subscription alone does not choose
that authority.

The proposed scope is storage-maintainer PRs. Source PR approvals and
compatibility bypass remain external; source CI still uses scoped OIDC requests,
without mirrored schema PRs. Accepted inputs, generated outputs, and receipts
remain together; gateway artifact selection, source ordering, fresh composition,
conditional acceptance, and original-receipt recovery retain their existing
contracts.

## Events and command authority

GitHub Apps can subscribe to selected events through an HTTPS webhook endpoint.
[App webhooks](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/using-webhooks-with-github-apps).

| Event | Relevant activity | Documented subscription permission |
| --- | --- | --- |
| `pull_request` | PR lifecycle and head changes | Pull requests: read |
| `pull_request_review` | Submitted, edited, or dismissed reviews | Pull requests: read |
| `issue_comment` | Ordinary conversation comments on issues **and PRs** | Issues: read |
| `pull_request_review_comment` | Comments on particular lines of the PR diff | Pull requests: read |

These are distinct subscriptions. An ordinary command in the PR conversation
uses `issue_comment`, not `pull_request_review_comment`. The issue object's
`pull_request` property distinguishes a PR comment from an issue comment.
[Webhook event reference](https://docs.github.com/en/webhooks/webhook-events-and-payloads),
[Comment and review activity](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#issue_comment).

A valid `X-Hub-Signature-256` authenticates GitHub's delivery and its body. It
does not establish that the person who wrote a command has publication
authority.
[Delivery validation](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries).
GitHub exposes a collaborator's effective repository permission and role through
the permissions API; installation tokens can call it with Metadata: read.
GitHub also recommends durable numeric user/repository IDs instead of mutable
names.
[Repository permissions](https://docs.github.com/en/rest/collaborators/collaborators#get-repository-permissions-for-a-user),
[App authorization guidance](https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/best-practices-for-creating-a-github-app#check-authorization-thoroughly-durably-and-often).

**Design inference:** a command handler must verify the installation and storage
repository, identify the PR and its expected base, check the command actor under
the accepted authority policy, and bind the intent to exact PR inputs. A comment
body, display name, or `author_association` label is not a substitute for that
policy. Ordinary comments identify the issue and author, rather than binding a
review to a Git commit; a command should explicitly identify the intended head,
or use another unambiguous confirmation protocol. Fetching whichever head is
current when a delayed webhook arrives does not establish which bytes the
commenter intended.
[Issue comment representation](https://docs.github.com/en/rest/issues/comments#get-an-issue-comment).

A narrowly scoped first workflow could process only newly created merge/retry
commands and treat subsequent edits as neither new commands nor implicit
authorization. PR/review events could refresh status or invalidate an attempt.
Automatic merging whenever requirements become satisfied is a different
workflow: it must also account for check completion and other eligibility
changes, and recover missed notifications. A rejected request must not silently
become standing background publication work under the current retry contract.

## Delivery is not acceptance

GitHub requires a 2xx response within ten seconds. Its documentation suggests
asynchronous processing when necessary. A requested redelivery retains the same
`X-GitHub-Delivery` identifier.
[Webhook delivery practices](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks).
Failed deliveries are not retried automatically; GitHub.com currently exposes
redelivery for the previous three days. GitHub documents a scheduled recovery
process that lists failed App deliveries and requests redelivery.
[Redelivery window](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/redelivering-webhooks),
[App recovery example](https://docs.github.com/en/webhooks/using-webhooks/automatically-redelivering-failed-deliveries-for-a-github-app-webhook).
Events may arrive out of order or several minutes late.
[Delivery ordering and delays](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/troubleshooting-webhooks#webhooks-deliveries-are-out-of-order).

**Design inference:** acknowledge receipt separately from the eventual
publication result. Before acknowledging work that must survive a crash, retain
it durably or provide a reconciliation path that can recover it from GitHub.
The storage mechanism is not selected here. Deduplicate deliveries, but also
deduplicate the stable publication intent: multiple distinct events can concern
one attempt, and a retry after lost acceptance must recover the original
receipt. A delivery ID alone is not the publication identity.

Terminal rejection also needs a durable outcome or equivalent recovery rule.
Rescanning an open PR and its old merge command after restart must not turn a
previous rejection into another publication attempt. Git acceptance receipts
cover successful attempts; they do not by themselves record failed ones. A new
explicit retry can authorize another attempt while still checking for the
original acceptance first. This is an additional lifecycle obligation for event
intake, not a decision to add a particular database or queue.

Re-read the relevant PR state before acting on an event. Do not process delivery
order as the order of authorized control changes, or use timestamps instead of
the accepted source-ancestry ordering. Delivery recovery resumes transport or an
interrupted attempt; semantic rejection still requires an explicit corrected or
retried request. An outage beyond GitHub's delivery window needs reconciliation
from retained work/PR records or an explicit caller retry.

## Restricting accepted writes to the App

Rulesets can restrict updates so only bypass actors may update a matching branch;
GitHub Apps can be those actors. A PR-only bypass does not allow direct pushes.
Separate applicable rulesets and branch protections still combine, with the
most restrictive applicable requirements enforced.
[Update restrictions](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#restrict-updates),
[App bypass modes](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#granting-bypass-permissions-for-your-branch-or-tag-ruleset),
[Rule layering](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets#about-rule-layering).

Classic branch protection can also restrict pushes to selected Apps, but allowed
push actors remain subject to required checks and PR requirements; administrative
access has additional exceptions. Merely installing an App with Contents: write
does not establish an exclusive writer.
[Protected-branch restrictions](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#restrict-who-can-push-to-matching-branches).

**Design inference:** an App-only accepted writer is an externally configured
operational boundary, including direct schema publications, maintainer merges,
other automation, and emergency writes. Repository administrators can still
change that configuration. Restricting writers and granting review bypass are
separate decisions. The selected rules must permit the existing direct schema
publication path; requiring a new storage PR for every schema update would
contradict the accepted workflow.

In particular, a blanket required-PR rule on that branch needs a compatible
bypass for the direct publisher. If the same credential can bypass control
reviews, repository configuration alone does not prevent it from doing so.
The asynchronous merge API's `bypass_rules: false` offers an explicit way to
request rule evaluation, but using the appropriate path is then part of the
service contract; the input does not select particular rulesets to bypass.
Interaction with the chosen App-only writer restriction remains untested.
The distinct `exempt` ruleset mode means rules are not run for that actor and
must not be mistaken for enforced reviews.
[Merge bypass option](https://docs.github.com/en/rest/pulls/pulls#merge-a-pull-request-asynchronously),
[Ruleset bypass modes](https://docs.github.com/en/rest/repos/rules#create-a-repository-ruleset).

If control merges require a credential unable to bypass review requirements,
separate App identities/grants for source publication and control merging are
an alternative to validate. Making every source publication a storage PR would
instead change [ADR 0008](../adr/0008-source-prs-own-schema-approval.md). Neither
additional mechanism is selected here.

## Two ways for Selvedge to finish a maintainer PR

### Native merge after preparing the complete candidate

REST merge accepts an expected PR-head `sha`; GraphQL exposes
`expectedHeadOid`. Neither input has an expected base OID or a caller-supplied
merge tree. The asynchronous REST API additionally exposes a direct-merge
choice and an explicit `bypass_rules` option. Its initial acceptance does not
mean merge rules have passed or the PR has merged; its result expires after
24 hours.
[REST merge operations](https://docs.github.com/en/rest/pulls/pulls#merge-a-pull-request),
[GraphQL merge input](https://docs.github.com/en/graphql/reference/pulls#mergepullrequestinput).

**Design inference:** native merge can preserve the required complete snapshot
if Selvedge first puts the exact validated inputs, outputs, and receipt into
the PR candidate, then prevents the accepted base from changing until that
candidate is merged. This requires serialization across **all** accepted writes,
including source publications, and recovery of an unknown merge result before
another writer proceeds. A mutex in one process does not establish that
guarantee across workers, restarts, or a still-running hosted merge. The receipt
format must also accommodate GitHub determining the final merge commit OID.

This is a proposed substitute for the target-head equality condition in the
native operation, not a newly discovered CAS parameter. It needs explicit
agreement and host validation if chosen; the accepted conditional-write
contract should not be silently weakened. Directly installing a prepared commit
can retain that condition without replacing it with exclusive execution.

The review cost remains real. Stale-review dismissal can invalidate approvals
when the diff changes; latest-push approval can require another reviewer after a
bot update. Manual merge commits can also be rejected if their contents differ
from GitHub's expected merge under those settings.
[Review freshness](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-a-pull-request-before-merging).

**Consequence:** either hold up other publications while the final generated
candidate receives required approval, or release the writer and potentially
rebuild/reapprove when the base moves. A policy that trusts bot-generated
changes without renewed approval can reduce that loop, but changes what native
review attests to. Subscribing to review events does not remove this choice.

### Selvedge finalizes the authorized control proposal

GitHub can create a commit with a chosen tree and parents, and GraphQL
`updateRefs` can require the accepted ref's previous OID. This lets Selvedge
reconstruct against head `H`, validate under accepted controls, retain the
matching outputs and receipt, and conditionally accept the complete tree.
[Commit creation](https://docs.github.com/en/rest/git/commits#create-a-commit),
[Conditional ref updates](https://docs.github.com/en/graphql/reference/git#updaterefs).

**Design inference:** an authorized command bound to immutable PR inputs can
enter the existing finalizer. Independent schema advancement causes fresh
composition and an internal CAS retry; changed control assumptions require
conflict handling rather than restoration of an obsolete desired value.
Approval authority must come from the agreed command/admission contract.
Reading GitHub review state immediately before a ref update does not make
those two operations atomic. This option therefore needs an explicit decision
about delegation and later review withdrawal or PR closure.

Including the original PR head as a parent can preserve PR ancestry. GitHub
documents indirect merge recognition when head commits become reachable from
the base, even when that PR's protection requirements were unsatisfied.
It does not document synchronous recognition as part of `updateRefs`.
[Indirect merges](https://docs.github.com/en/pull-requests/reference/pull-request-merges#indirect-merges).
The accepted receipt remains authoritative; ancestry or a merged badge alone
does not prove approved contents. Recognition on the actual chosen branch and
event timing remain untested. Proposal ancestors must not be mistaken for
complete accepted snapshots.

## Recommended workflow for discussion

Prefer the second option, with one accepted branch and an explicit command such
as `/selvedge merge <head-sha>`. PR activity can refresh checks; a permitted
maintainer's command is the final authorization for the identified control
change. That command model is a proposed delegation of storage authority, not
an assertion that native required-review rules are enforced by a custom write.
It leaves source-repository approval and compatibility exceptions external.

For example, a maintainer PR changes composer version 1 to version 2. After
review, its authorized command enters the same finalizer as source publication.
The finalizer preserves the latest accepted subgraph schemas, applies the exact
control change under existing governing controls, evaluates it, and generates
the complete candidate. It then conditionally installs the snapshot and receipt.
It may include the original PR head as the second parent, subject to the
acceptance-history and host-recognition conditions above. Generated bytes need
not be pushed back into the proposal before each contention retry.

The distinction between two kinds of intervening change remains necessary:

- A new source schema causes reconstruction and recomposition. It does not
  change the requested composer update.
- An accepted control change may invalidate the approved proposal's assumptions.
  A conservative initial rule is to stop if the expected maintained-control
  identity changed and require a refreshed proposal/authorization. This is a
  recommendation for the next decision after selecting storage authority; it
  deliberately trades more retries for avoiding implicit control-change merges.

A genuine fresh validation failure ends the attempt and keeps the PR pending.
An uncertain write is reconciled before reporting failure or trying again.
Acceptance is the retained snapshot receipt; PR reporting follows that result.
Later PR closure or review withdrawal cannot be promised to revoke an admitted
command atomically with a Git ref update. The command's submission and
cancellation semantics therefore remain an explicit decision.

This removes the separate reviewed-control history, cumulative control
publication ordering, and a control PR merged before its update takes effect.
It adds event recovery and command authorization to the service. These are the
concrete tradeoffs behind preferring this workflow; the alternative with native
merge enforcement remains available if that approval guarantee is required.

## Remaining decisions

1. **Storage authority:** must GitHub evaluate its live review requirements in
   the final native merge, or may an authorized maintainer command delegate
   finalization of exact control inputs to Selvedge? This does not transfer
   source PR approval or compatibility-exception policy to Selvedge.
2. **Trigger and lifetime:** use an explicit merge/retry command, or automatic
   eligibility events? For an explicit command, define who may issue it, which
   immutable inputs it authorizes, and whether subsequent closure or review
   withdrawal cancels an in-flight attempt. Continuous revocation is not
   atomically guaranteed by reading PR state before CAS.
3. **Completion:** is acceptance established by the durable snapshot receipt,
   with PR status subsequently reconciled, or is native GitHub merge completion
   itself required? The latter selects the complete-candidate and serialization
   work above.

The one-branch finalizer offers the simpler acceptance lifecycle if storage
authority is deliberately delegated: a control PR remains pending until a
complete snapshot is accepted, and a rejected candidate leaves accepted controls
unchanged. Its cost is webhook processing/recovery and an explicit storage
authorization contract. Native merging retains GitHub's live merge enforcement
but adds candidate updates, review freshness, and durable serialization. Neither
requires mirrored source PRs or changes explicit gateway activation.

Before an implementation can claim host compatibility, exercise the chosen App
and real rules with authorized/unauthorized commands, edited or stale heads,
review withdrawal, duplicate/reordered/missed deliveries, concurrent schema and
control publications, crash/response-loss recovery, and exact receipt/output
retrieval. For native merging, also test bot-push review behavior and an
unresolved hosted merge during restart. For custom acceptance, test branch-rule
admission and PR recognition. These checks have not been performed.
