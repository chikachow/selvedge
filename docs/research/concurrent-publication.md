# Concurrent subgraph publication

Research date: 2026-11-01. The user requires valid independent changes to publish
successfully even when they overlap in time. This refines
[ADR 0009](../adr/0009-snapshot-checks-and-conditional-registry-updates.md): a moved
registry head is internal contention, not by itself a semantic conflict exposed
to the caller. The recommendations below preserve the original evaluation. No
implementation or concurrency experiments were run for this note.

Subsequent decision: [ADR 0010](../adr/0010-retain-validated-outputs-with-acceptance.md)
retains matching generated output and receipt in the acceptance commit. The
earlier post-acceptance artifact-production sequence at the end of this note is
superseded. Retry/rejection and source-ordering contracts were subsequently
accepted in [ADR 0011](../adr/0011-caller-owned-retries-and-publication-receipts.md)
and [ADR 0012](../adr/0012-order-publications-by-source-history.md).

## Apply the update to current state

Recommended publication loop:

1. Read registry head H and its governing configuration.
2. Apply only the authorized subgraph SDL/provenance replacements to H, retaining
   every other subgraph and maintained configuration.
3. Validate and compose the complete candidate; compute the associated reports.
4. Conditionally commit against H. If the head moved, repeat with the new state,
   including scope and source-order checks.
5. On composition failure, check the head again. If it moved, reevaluate before
   rejecting; otherwise return diagnostics explicitly associated with H.

The whole graph is not replaced by the caller's earlier snapshot. If A publishes
one subgraph and B publishes another, B's refreshed candidate retains A. Both
succeed if the combined graph composes. The publication report records the
revision actually used, even when it differs from the source PR's earlier check.

GitHub supports the conditional-write boundary through `expectedHeadOid` on
`createCommitOnBranch`. This documents the primitive, not a verified integration
with the intended token permissions or repository protection rules.
[Conditional commit input](https://docs.github.com/en/graphql/reference/commits#createcommitonbranchinput).

## Separate semantic failure from operation failure

Recommendation for an evaluated uncomposable candidate: reject that request
without changing the accepted registry or publishing artifacts for it. Return
composition diagnostics, involved subgraphs/coordinates, submitted source
identities, and the evaluated registry revision. The caller can correct the
schema or explicitly retry when a prerequisite changes. The request is not
accepted into a queue for automatic later publication.

If concurrent updates are mutually incompatible, the first accepted update stays
accepted and the later candidate may fail against it. No response establishes
that a candidate can never become valid against a future registry revision.

Repeated head movement consumes an operation budget. If that budget or an
infrastructure limit is reached, report a retryable operational outcome rather
than inventing a composition error. A timed-out Git write may already have
succeeded; reconcile it before replaying or asserting that nothing was written.

Compatibility findings remain distinct from composition validity. Under the
[accepted exception boundary](../adr/0001-repository-managed-compatibility-exceptions.md),
an authorized publisher does not need Selvedge to grant an exception. Fresh
breaking findings stay visible in the report; they do not silently become
compatible or introduce a new service-owned approval system.

## Same-subgraph writes need a different rule

Git-head conditional updates prevent lost concurrent registry writes, but cannot
recognize whether one source version is older than another. An older schema may
still compose and may even pass structural compatibility checks. Likewise, a
simple expected-subgraph-version can reject a valid newer descendant merely
because an intermediate version published first.

Recommended ordering, pending confirmation: advance according to source history.
A newer descendant can publish after recomposition; an older ancestor that has
already been overtaken is superseded; divergent or rewritten history requires
an explicit recovery contract. Intentional rollback can be a newer source commit
that restores the earlier schema. Timestamps are not a substitute for source
ordering evidence.

Source progress matters even when SDL bytes match: remembering only schema
digests can lose the fact that a later source revision deliberately restored
earlier SDL. The exact representation belongs to the eventual publication
contract; no source repositories are available yet to validate it.

## Retries and coordinated changes

Identify publication intent independently of the registry baseline that may
change during internal retries. Bind that identity to the exact requested
updates and provenance. An already accepted request should return its original
acceptance, even if a later publication superseded it; replaying it must not
restore old SDL. Git acceptance history can retain evidence for recovery without
assuming a service-owned job queue. This is a proposed contract, not a claim
that GitHub request IDs provide native deduplication.

For interdependent changes that compose only together, submit one complete
batch with authority for every replacement. This preserves the existing
multi-subgraph requirement while keeping the accepted graph composable after
each registry update. Automatically accumulating invalid partial updates would
introduce another persistent state and lifecycle to manage.

Acceptance identifies an exact registry commit. Immutable artifact production
follows it and may require its own recovery. Acceptance, artifact availability,
and gateway activation remain distinct; completion order does not select an
active gateway revision.
