# System design review against agent instructions

Review date: 2026-11-01. Request: apply the repository's agent instructions to
the current system design. This is a static design review and revision, not an
implementation or a new approval of pending contracts. The starting point is the
design after the storage-PR event proposal; Q27 remains unanswered.

The governing review criteria are in [AGENTS.md](../../AGENTS.md). Accepted
decisions remain in [the ADRs indexed by the design](../design.md#accepted-decisions),
and domain language remains in [GLOSSARY.md](../../GLOSSARY.md). This record
captures evaluation history; the [current design](../design.md) owns the revised
architecture and open frontier.

## Assessment

Retain the shared checker and conditional publication operation. They serve
required behavior across source checks, source publication, and maintainer
changes. Their concrete benefit is locality: callers do not each reconstruct
the complete graph, resolve contention, classify uncertain acceptance, or
recover an original receipt.

The accepted contracts are coherent at this level. The design is not yet an
implementable integration specification: important input and authority
relationships remain unresolved. A smaller branch diagram alone cannot settle
those relationships. There is no application source or behavioral test suite in
the repository, so findings concern design coverage and contract questions, not
demonstrated runtime defects.

## Findings and disposition

| Classification | Trigger, consequence, and evidence | Disposition |
| --- | --- | --- |
| Unmet documentation outcome | The handoff requires bounded input, time, and resource consumption and forbids privileged execution of untrusted proposal code. The prior current-design requirements mentioned trusted tools and policy but did not carry the resource obligation into the architectural responsibilities. An implementation could therefore omit it while appearing to follow the short design. | Restored the carried requirement and assigned immutable input capture, request validation, and trusted-tool execution to their owning boundaries. Concrete budgets and execution arrangement remain implementation choices. |
| Incomplete lifecycle description | A release upload can fail after acceptance. ADR 0010 retains outputs in Git, but callers still need to distinguish accepted state from distribution failure. A single undifferentiated failure could encourage replay or regeneration. | Made acceptance, unverified writes, rejection, and release availability explicit in the shared lifecycle and caller outcomes. No new retention policy was added. |
| Contract question | A post-merge build can generate different SDL from the checked PR build; authenticating its job and recomposing its schema does not establish the earlier check's relationship to those bytes. The prior design listed correspondence alongside routine formats and retrieval work. | Moved checked-versus-published correspondence to the material decision frontier. Did not invent a mandatory attestation, byte-equality rule, or new approval gate. |
| Contract question | A different intent may submit different SDL for the same source revision. ADR 0011 detects changed payload under the same intent and inconsistent outputs for identical declared generation inputs; ADR 0012 orders revisions. Neither alone defines a legitimate different generation. | Exposed generation identity and permitted same-revision changes as a material contract decision. Serialization remains routine once the contract is chosen. |
| Conditional design tradeoff | The Q27 command proposal removes a review branch but changes the admission mechanism. Native live review evaluation, immutable-command authorization, withdrawal, and terminal-rejection recovery are not interchangeable. | Kept the one-branch recommendation conditional and included its event/command lifecycle costs. No command authority, bypass setting, or cancellation contract was accepted. |
| Dependent contract questions | A stale maintained-control update can overwrite a newer approved value, and introducing/removing a subgraph may need both maintained configuration and source schema changes. A generic change payload cannot establish either authority. | Kept control preconditions, authorization lifetime, and coordinated membership changes explicit; their admission path depends on the storage choice. |

The handoff was supplied in the conversation. The relevant resource and
privilege requirements are now carried in
[current requirements](../design.md#requirements-carried-forward), rather than
depending on a personal attachment path. Detailed existing evidence for the
other findings lives in [request authentication](../research/request-authentication.md),
[control finalization](../research/control-pr-finalization.md), and
[storage PR events](../research/storage-pr-events.md).

## Alternative interfaces considered

Three independent agents examined the whole concern under different constraints.
Each received the user request, repository instructions, glossary, current
design, and accepted ADRs, without a prescribed verdict. Their analyses were
static; they performed no implementation or host experiments.

### Complete lifecycle operations

Expose checking, publication, and exact accepted-artifact retrieval. Hide
candidate assembly, source ordering, conditional writes, receipt lookup, and
recovery inside the publication module. Use the same graph checker through the
CLI and service.

This gives callers substantial leverage without removing meaningful outcomes
from the interface. Its cost is concentrated implementation responsibility:
changes to the shared publisher must preserve every admission variant. Three
method names alone do not make the semantic interface small.

### Explicit preparation and acceptance stages

Expose prepare, assess, accept, recover, distribute, retrieve, and comparison
operations with transferable prepared configurations. This supports detailed
orchestration, inspectable intermediate results, and future durable drafts.

The cost is lifecycle knowledge spread across callers: stale baselines,
candidate retention, trusted versus supplied reports, output reuse, and uncertain
acceptance. The accepted release needs complete local checker inputs, but has no
demonstrated need for public preparation handles or a draft inventory. Retain
immutable prepared data internally without exposing a new workflow protocol.

### Source-CI convenience interface

Put a small submission client beside source generation. Its check and publish
operations handle OIDC transport, stable intent submission, report association,
and understandable outcomes. Source repositories still own generation,
scheduling, and explicit retries.

This reduces repeated integration work in ordinary callers. Its cost is a
client/service protocol and versioned client delivery. It must not become a
mandatory centrally approved build pipeline or move the publication algorithm
back into every source repository. It does not resolve maintainer authority.

### Recommendation and cost comparison

Use complete lifecycle operations with internal prepared data and a thin
submission client. Keep the graph checker independent of GitHub. The advantage
is shared lifecycle behavior and ordinary source-CI ergonomics; an extra public
stage or deployed module needs a concrete caller benefit.

| Concern | Recommended design | Why retain or avoid the alternative |
| --- | --- | --- |
| Publication | One owner of assembly, evaluation, conditional installation, and reconciliation | Splitting these into caller-managed stages preserves the hard work while spreading ordering and recovery obligations |
| Candidate identity | Immutable internal values and ordinary checker files | Public handles or branches need a demonstrated durable handoff/drafting requirement and its lifecycle |
| Caller integration | Thin client around complete operations | A reusable client reduces repeated protocol work; a mandatory centrally governed generator would change the trust contract |
| Storage and execution | Git accepted history and bounded checker execution, with narrow host/engine adapters | Add a database, worker service, cache, or index for demonstrated recovery needs or measured workload costs; count of deployed pieces is not the objective |
| Storage PR workflow | Conditional preference for one accepted branch | Native review and delegated commands differ in authority and recovery; branch count does not decide between them |

Existing source approval, externally managed compatibility exceptions, explicit
activation, bounded static assurance, source ordering, atomic batches, and
retained outputs remain intact. No domain terms were renamed and no ADR was
superseded. These refinements have documentation and eventual implementation
costs, but do not require a data migration because no application exists yet.

## Verification boundary

The appropriate future evidence crosses each complete operation. This is a
verification plan derived from existing contracts, not a new standing coverage
threshold or approval policy.

| Guarantee or failure scenario | Independent evidence to obtain |
| --- | --- |
| Complete Federation validity and public API classification | Real engine fixtures with expected success/failure and classifications derived from supported GraphQL/Federation behavior; compare the chosen composition integration with the supported Apollo tool and actual gateway |
| Operation evidence remains honest | Fixed baseline corpus exercised against both schemas; absent, empty, invalid, and unreadable inputs; deleting a proposed document must not erase a baseline regression |
| Authorized immutable input | Wrong principal/scope and changed payload tests at ingress; mutation of supplied material after capture cannot change evaluated or accepted bytes; no publication token in checker inputs/helper environment |
| Current admission controls | Ownership/policy changes during contention; reconstruction must apply current accepted grants without allowing proposed grants to authorize themselves |
| Independent concurrent source updates | Two publications from one head, then combined validation; both updates survive when valid, and a combined invalid graph is not accepted |
| A rejecting evaluation becomes stale | Candidate is invalid at H but valid after H2 is accepted during evaluation; the finalizer must observe the head again and reevaluate. Failure to read the head is operational failure, not definitive semantic rejection |
| Same-subgraph and batch ordering | Newer source acceptance followed by delayed older intent; unchanged-SDL source progress; a batch with one overtaken participant cannot publish a reduced batch |
| Lost acceptance response | Accept A, advance with B, then retry A; recover A's original receipt without restoring its old SDL. Unknown write status cannot be reported as definitive rejection |
| Retained output and distribution | Fail upload after acceptance and retry distribution; retrieve identical retained bytes and preserve the original acceptance identity |
| Source check association and publication grants | Exercise actual PR, merge, rerun, and any supported fork paths with the configured App/rules and OIDC claims; local Git tests cannot establish these host guarantees |
| Selected maintainer integration | Real chosen credentials/rules, changed PR inputs, review withdrawal/closure, control contention, and interrupted completion. For event intake, redeliver a terminally rejected command without reviving it |

No tests are removed or consolidated: there are none for the application yet.
Do not claim regression detection until unchanged expectations have failed
against the faulty behavior and passed after its repair. Local Git experiments
already recorded in [the Git study](../research/git-state-model.md#verification-boundary)
remain evidence about local primitives, not proof of hosted approval enforcement.

This revision is checked through complete document-diff review, local link and
anchor validation, and whitespace checks. The repository has no configured
Markdown formatter. Host compatibility, performance, real-schema composition,
and deployed behavior remain untested. Implementation and any eventual
merge-readiness claim remain subject to the existing repository gates.

Two of the independent agents subsequently reviewed the complete revision
against the original handoff, accepted ADRs, and repository instructions. Both
identified the same gap in the proposed rejection path: conditioning refresh on
already knowing that the head advanced did not ensure a failed evaluation ever
observed that advancement. The lifecycle now explicitly rereads the accepted
head before rejection, and the plan includes a candidate invalid at H but valid
at concurrent H2. Both reviewers verified the correction and reported no
remaining material findings. An optional distribution-retry ownership
clarification was also incorporated. These are static review results, not an
observed faulty-behavior failure and restored pass.
