# Authentication for explicit CI requests

Research date: 2026-11-01. This supports the request-driven service in the
[design discussion](../design.md) and the
[source approval boundary](../adr/0008-source-prs-own-schema-approval.md).
The accepted assurance boundary is authenticated submissions and an exact binding
between accepted inputs and generated artifacts. Source/build-to-SDL linkage
remains a submitting party's claim unless separately verified; mandatory build
attestations are outside the initial scope. Recommendations below do not establish
additional approval requirements.

No concrete subgraph repositories exist yet. This is primary-source research and
static design analysis, not a tested GitHub integration. No tokens were requested,
host settings changed, or workflows executed.

Subsequent decision: Q28 accepted Actions OIDC for initial CI requests in
[ADR 0014](../adr/0014-authenticate-ci-requests-with-actions-oidc.md). Detailed
trust configuration and the unperformed integration checks below remain work
for the eventual source repositories.

## Accepted initial mechanism

Use GitHub Actions OIDC for incoming CI requests, with separate authorization for
checking and publication. GitHub issues job identity tokens that a relying
service can validate; this avoids distributing a long-lived Selvedge credential
to every source repository. The service still needs an explicit trust policy.
[GitHub OIDC overview](https://docs.github.com/en/actions/concepts/security/openid-connect).

| Option | Benefit | Cost and limitation |
| --- | --- | --- |
| Actions OIDC | Authenticated job context without a per-source static secret | GitHub-specific verification and claim policy; issuer keys and token availability become integration dependencies |
| Scoped static credential | Supports a generic CI client with a simple credential exchange | Secret distribution, rotation, and revocation; possession alone supplies no independently authenticated run, event, ref, or workflow identity |

The comparison is a design assessment. A static credential could meet the
accepted submission-authentication boundary if deliberately scoped, but there is
no demonstrated non-Actions client requirement that justifies maintaining both
mechanisms initially. Incoming OIDC and the service's outbound GitHub App
credentials serve different purposes.

## What GitHub authenticates

GitHub.com's discovery document currently specifies issuer
`https://token.actions.githubusercontent.com`, its JWKS endpoint, `RS256`, and the
supported claims. Verification must use this configured issuer's keys, rather
than trusting a key location supplied by an incoming token.
[Live issuer metadata](https://token.actions.githubusercontent.com/.well-known/openid-configuration),
[JWT issuer and algorithm validation](https://www.rfc-editor.org/rfc/rfc8725#section-3).

| Claim | Documented meaning |
| --- | --- |
| `aud` | Intended audience; the requester can select a custom value |
| `repository_id`, `repository_owner_id` | Repository and owner IDs for the workflow context |
| `event_name`, `ref` | Triggering event and ref |
| `workflow_ref`, `workflow_sha` | Workflow path/ref and workflow-file commit |
| `job_workflow_ref`, `job_workflow_sha` | Called reusable workflow path/ref and commit, when applicable |
| `run_id`, `run_attempt`, `check_run_id` | Run, retry attempt, and current job's check-run identifiers |

These are GitHub's claim definitions, not a Selvedge authorization policy.
[OIDC claim reference](https://docs.github.com/en/actions/reference/security/oidc#oidc-token-claims).

The discovery document also advertises `sha`, and GitHub's token examples include
it. Keep that signed value distinct from the claimed actual build revision and
PR head. GitHub's Actions context defines `github.sha` according to the triggering
event; neither a context value nor a token inspects a later checkout override.
Verify the event-specific correspondence in the actual integration before using
it to establish a source relationship.
[Supported claims](https://token.actions.githubusercontent.com/.well-known/openid-configuration),
[Token example](https://docs.github.com/en/actions/concepts/security/openid-connect#understanding-the-oidc-token),
[Actions context](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts#github-context).

Reusable-workflow tokens retain the caller's context and identify the called
workflow separately. Checking only the reusable workflow identity would omit
which source repository invoked it; checking only the caller would omit which
reusable implementation ran.
[Reusable-workflow OIDC](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-with-reusable-workflows#how-the-token-works-with-reusable-workflows).

Do not hard-code the historic name-only `sub` format. GitHub documents an
ID-containing default for repositories created after 2026-07-15, and subject
customization is supported. Prefer explicit claim conditions for the authority
being granted, while validating the issuer/subject association.
[Subject formats](https://docs.github.com/en/actions/reference/security/oidc#immutable-subject-claims),
[Subject validation](https://www.rfc-editor.org/rfc/rfc8725#section-3.8).

## Proposed authorization policy

Authenticate at the service boundary before consuming authority-bearing fields.
Require the configured issuer, allowed signing algorithm, intended service
audience, and valid token times; validate required claim types and reject missing
policy inputs. An audience names the recipient and is not itself a publishing
grant. JWT standards require audience validation and define expiry/not-before
processing; local clock tolerance and maximum accepted token age are application
configuration.
[JWT best current practices](https://www.rfc-editor.org/rfc/rfc8725#section-3),
[JWT time claims](https://www.rfc-editor.org/rfc/rfc7519#section-4.1).

Registry-maintained grants should map authenticated repository/owner IDs and
workflow conditions to an explicit graph, permitted subgraphs, and operation.
Names remain useful display metadata. Matching both owner and repository IDs
also makes continued authority after transfer an explicit configuration choice.
Do not authorize all subgraphs merely because a caller belongs to the organization.
Batch publication needs authority for every included subgraph; one participant's
token does not automatically authorize another participant's replacement.

| Operation | Recommended initial grant |
| --- | --- |
| Check | Configured source workflow and supported proposal events; only permitted subgraph replacements; registry remains unchanged |
| Publish | Configured trusted source workflow on its accepted branch, initially a `push` context; only permitted schema/provenance updates |

Reject proposal-event credentials at the publication endpoint even when they can
request checks. Apply the scope to parsed updates, not just a claimed subgraph
name; policy, routing, and ownership remain registry-maintainer inputs.

The least additional machinery consistent with the accepted delegation is to
trust the configured source publication workflow under that repository's
controls. Match its path/ref and event, and record its resolved workflow SHA.
Requiring a separately controlled reusable publisher pinned to a SHA is a
stronger alternative: it constrains whose workflow edits can change publisher
behavior, but adds shared workflow ownership and upgrade coordination. A reused
workflow can be pinned by commit; a same-repository relative call uses the
caller's commit. Neither arrangement proves the truth of arbitrary submitted SDL.
[Reusable workflow references](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows#calling-a-reusable-workflow).

OIDC fetching requires `id-token: write`. Set permissions at the submitting job
and keep its other permissions minimal. A called workflow cannot elevate the
caller's `GITHUB_TOKEN` permissions; cross-organization OIDC calls also require
an explicit caller grant. These permissions enable token acquisition, while
Selvedge determines the token's authority at its API.
[Workflow permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#permissions),
[Reusable workflow permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations#supported-keywords-for-jobs-that-call-a-reusable-workflow),
[OIDC caller permissions](https://docs.github.com/en/actions/reference/security/oidc#reusable-workflows).

A trusted reusable workflow must still treat caller inputs and downloaded files
as data. Invoking a pinned workflow does not make an arbitrary command input,
checkout, dependency, or build script trusted. In particular, do not obtain
publication privileges by executing untrusted PR code in `pull_request_target`.
GitHub documents that event's elevated trust and the resulting code-execution
risk. Ordinary fork-PR permissions can restrict token acquisition; support for
fork submissions must be established with the eventual repository settings.
[PR event security](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target),
[Fork permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#changing-the-permissions-in-a-forked-repository).

## Revision identity and source approval

For an ordinary open PR, the default checkout and `GITHUB_SHA` identify the
synthetic merge revision; the PR head is available separately. A `push` workflow
instead identifies the pushed ref and tip. The workflow can explicitly check out
different code, so record the PR head, event revision, claimed build revision,
and received SDL digest separately.
[PR and push event semantics](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows).

Inference: a valid `push` token on an accepted branch does not prove that a PR was
merged or that particular reviewers approved it. Authorized direct pushes also
produce that context. Source-repository rules own admission to that branch and
compatibility exceptions; Selvedge must not reconstruct an additional approval
system from actor names, review membership, or the token. A named branch or an
environment label is likewise not proof of deployment readiness.

The service can compare request revision fields to authenticated context and,
where necessary, retrieve the identified run/PR during the request. That can
verify GitHub relationships without proving which bytes a build generated.
GitHub exposes individual run and PR retrieval; any required read permissions
and race handling must be established in integration.
[Get a workflow run](https://docs.github.com/en/rest/actions/workflow-runs#get-a-workflow-run),
[Get a pull request](https://docs.github.com/en/rest/pulls/pulls#get-a-pull-request).

Such reads do not require service-owned webhook subscriptions, PR inventory,
automatic rechecks, or background publication. A new token from an old run does
not make its source revision newer. Token validity, source ordering, and whether
publication must match an earlier checked SDL are separate contracts.

## Authoritative source checks

The service's GitHub App can create a check for an explicit `head_sha` with
Checks write permission. Branch protection and rulesets can require a particular
App as the expected producer. Configure that identity for Selvedge's distinct
validity and compatibility results; a matching check name alone is insufficient.
[Check Runs API](https://docs.github.com/en/rest/checks/runs#create-a-check-run),
[Expected App identity](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#require-status-checks-before-merging),
[Ruleset expected identity](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-status-checks-to-pass-before-merging).

Inference: choosing the shared GitHub Actions App does not identify one trusted
workflow. A dedicated Selvedge App gives the reporting service an enforceable
producer identity, but that service must constrain requested check targets to
the authorized repository and verified proposal association. Do not let a body
field direct the App to report success on an unrelated repository or revision.
Reports should state which received SDL and baseline were analyzed, without
implying independently verified build provenance.

Consequently, an authorized source job can submit valid SDL unrelated to its
source code and receive a genuine successful Selvedge result for those bytes.
The expected App identity does not close that deliberately unverified linkage.
Source controls own the generator's trust unless a stronger build-verification
contract is adopted.

GitHub distinguishes head and test-merge checks, and its eligibility restrictions
for checks created by workflow jobs do not apply to external App checks. Actual
PR/fork/merge-group association still needs testing against the chosen rules.
[Required-check evaluation](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks).

Keep outbound App credentials in the service. Installation tokens can be limited
to selected repositories and a subset of the installation's permissions. Scope
source check reporting and registry writing accordingly; incoming CI does not
need the service's Checks or Contents write credentials.
[Installation token scope](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app).

## Binding the request to accepted artifacts

The recommended application boundary preserves three different kinds of evidence:

1. **Authenticated context:** validated GitHub job claims and any independently
   retrieved GitHub relationships. Retain relevant audit fields, not a live
   bearer token.
2. **Submitted claims:** source/build-to-SDL linkage and any asserted deployment
   state. Label these as claims until a separate verification mechanism supports
   a stronger statement.
3. **Selvedge's result:** digests computed from the received SDL and the exact
   complete inputs actually consumed, together with their generated outputs and
   acceptance receipt under [ADR 0010](../adr/0010-retain-validated-outputs-with-acceptance.md).

Compute content digests from retained input bytes rather than accepting a caller's
digest as proof. Validate and compose from that same immutable materialization,
then retain its outputs with acceptance. OIDC authenticates who may submit; the
application creates the input-to-output binding. The standard GitHub claims do
not contain a Selvedge request-body digest. This conclusion follows from the
documented token interface, rather than an executed tampering experiment.
[GitHub token interface](https://token.actions.githubusercontent.com/.well-known/openid-configuration).

JWT expiry bounds when authentication may be accepted; `jti` identifies a token
and can support replay defenses. Neither automatically deduplicates publication
or binds a retried request body. A candidate recovery contract is a stable
publication identity scoped to its authorized target and bound to exact inputs:
the same identity and inputs recover the original receipt; conflicting inputs
fail visibly. A retry can authenticate with a fresh token without changing that
publication identity. This remains application protocol design, not a GitHub
idempotency guarantee or a reason to expire accepted artifacts.
[JWT time and identifier claims](https://www.rfc-editor.org/rfc/rfc7519#section-4.1).

Build attestations are a separate mechanism that can add signed artifact/build
relationships when verified under an appropriate policy. GitHub explicitly
distinguishes attested provenance from a guarantee that software is secure.
They are not required for the initial accepted boundary and would not establish
source approval or deployed-schema equivalence by themselves.
[Artifact attestations](https://docs.github.com/en/actions/concepts/security/artifact-attestations).

## Decisions and verification still needed

Actions OIDC is the accepted initial request authenticator. Actual repository
IDs, accepted refs, workflow paths, audience, permissions, and subgraph grants
are onboarding configuration once
source repositories exist. No concrete values are inferred here.

Mandatory centrally pinned publisher code, non-Actions/static-credential clients,
or a privileged fork-submission bridge would change the trust or support model
and need an explicit requirement. The agreed source-approval delegation does
not itself require any of them. Same-subgraph ordering and retry recovery follow
[ADR 0012](../adr/0012-order-publications-by-source-history.md) and
[ADR 0011](../adr/0011-caller-owned-retries-and-publication-receipts.md);
checked-versus-published input correspondence remains a separate design question.

Before claiming the integration enforces these guarantees, exercise rejection
of wrong issuer/audience/repository/scope, proposal-token publication, and changed
retry bodies; verify accepted input/output identity and fresh-token receipt
recovery. Observe the actual claims for chosen PR, fork, reusable, postmerge,
rerun, and merge-group paths, and verify that another App cannot satisfy the
configured Selvedge check. These are proposed behavioral checks, not executed
results or new standing approval gates.
