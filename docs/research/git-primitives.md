# Git objects and conditional publication

Research date: 2026-11-01. Current primary documentation was read; no hosted
mutation, retention experiment, or implementation was exercised. This note
records primitives and design implications, not new accepted requirements. It
supports [snapshot checks and conditional updates](../adr/0009-snapshot-checks-and-conditional-registry-updates.md)
and informed the later [complete-snapshot decision](../adr/0010-retain-validated-outputs-with-acceptance.md).
The accepted generation/storage direction is governed by that ADR; detailed
mechanisms below remain proposals unless separately agreed.

## Immutable contents and mutable references

Git objects are immutable. Their object IDs hash their type and contents. A blob
contains file bytes; a tree records names, entry types and child object IDs. A
commit records a root tree, parent commits, author/committer identities and
times, and a message. Consequently, equal file bytes can reuse a blob, equal
directory contents can reuse a tree, and different commits can identify the
same tree. Changing commit metadata or parents changes commit identity without
requiring changed graph inputs. A reference names an object and can move.
[Git data model](https://git-scm.com/docs/gitdatamodel).

Git already exposes operations that create blobs, trees and commits without a
checkout. The standard object model supplies the content relations needed for
graph snapshots; using it does not require implementing another Merkle store.
[Git objects and plumbing](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects).

Exact-byte identity needs an explicit ingestion convention. `git hash-object`
can apply attribute-based filters when a path is supplied; `--no-filters` hashes
the supplied bytes without those conversions, including newline conversion.
Standard-input hashing implies that behavior unless a path is supplied.
[Hashing behavior](https://git-scm.com/docs/git-hash-object).

**Design inference:** identify the checked graph by the closed set of files and
configuration actually consumed, together with the tool profile. An accepted
registry commit identifies a historical event; it is unnecessarily sensitive to
unrelated metadata when used as the sole composition cache key. Preserve exact
SDL bytes and fixed entry modes at ingestion. Content reuse is an optimization;
it does not establish composition or compatibility.

## Conditional ref updates

Local Git's `update-ref <ref> <new> <old>` changes the reference only when its
current value equals the supplied old object ID. Its transaction mode can
prepare multiple ref changes together. The manual also warns that concurrent
readers can observe a subset of those changes: an all-or-nothing write is not
automatically an atomic multi-ref snapshot for readers.
[Local reference transactions](https://git-scm.com/docs/git-update-ref).

Git transport supports an explicit `--force-with-lease=<ref>:<expected>` check
against the remote ref. The form with an explicit expected value avoids reliance
on mutable remote-tracking refs. A lease permits a non-fast-forward update when
the expectation matches, so append-only history still requires constructing a
child of the accepted head and enforcing that invariant. `git push --atomic`
requires server support; it fails when unsupported and cannot transact across
different remotes.
[Push conditions and atomic pushes](https://git-scm.com/docs/git-push).

**Design inference:** read head H once for each attempt, prepare and validate the
complete replacement tree relative to H, and condition acceptance on H still
being current. A failed condition means reload and recompose. Git's textual
merge success cannot substitute for Federation validity. One accepted ref
provides one place to decide which complete graph was accepted.

## GitHub has more than a fast-forward check

GitHub GraphQL `createCommitOnBranch` appends a commit and advances its branch.
Its required `expectedHeadOid` identifies the expected prior head. The mutation
accepts file changes rather than an already constructed root-tree object. GitHub
chooses author/committer information and signs the commit when supported. Its
documented `clientMutationId` is an identifier, with no documented durable
deduplication guarantee.
[Commit creation contract](https://docs.github.com/en/graphql/reference/commits#createcommitonbranch)
and [expected head input](https://docs.github.com/en/graphql/reference/commits#createcommitonbranchinput).

The REST update-reference endpoint accepts a new `sha` and `force`; it does not
document an expected-old-OID parameter. With `force: false`, it requires a
fast-forward. These are different conditions. For example, after a ref is
rewound from H to ancestor A, a candidate child of H remains a fast-forward
from A, while an equality check expecting H should reject. This example is a
deduction from the documented conditions, not an observed GitHub experiment.
[REST reference updates](https://docs.github.com/en/rest/git/refs#update-a-reference).

GitHub GraphQL also documents `updateRefs`: it creates, updates or deletes
multiple refs **atomically within one repository**. Each `RefUpdate` can supply
`beforeOid`, `afterOid`, and `force`; `beforeOid` constrains the old value, and
the zero OID represents an absent ref. Thus a prepared commit can be installed
with an explicit old-value condition, and it is incorrect to infer that GitHub
has no conditional multi-ref primitive. This mutation does not create objects.
[Atomic ref mutation](https://docs.github.com/en/graphql/reference/git#updaterefs)
and [per-ref conditions](https://docs.github.com/en/graphql/reference/git#refupdate).

GitHub's REST tree creation accepts a `base_tree` plus entries that override
specified paths. Without `base_tree`, the new tree contains only the supplied
entries; committing it makes omitted parent files disappear. Reading recursive
trees can return `truncated: true`, requiring further subtree reads.
[Tree creation and retrieval](https://docs.github.com/en/rest/git/trees).
The REST commit endpoint can create a commit from a tree and parent IDs.
[Commit objects](https://docs.github.com/en/rest/git/commits#create-a-commit).

**Design inference:** prefer the smallest hosted adapter that can establish the
exact validated tree and head condition. `createCommitOnBranch` may suffice for
ordinary file replacements. Prepared trees/commits plus `updateRefs` give more
explicit object control, at the cost of extra requests and intermediate
unreachable objects. Verify byte and mode behavior before choosing. Neither
alternative establishes a transaction with source repositories or release
assets. Multi-ref capability alone is not a reason to add per-subgraph branches.

## Reachability is part of retention

Git garbage collection retains objects reachable from branches, tags and other
refs, and can retain additional objects through the index and reflogs. Reflogs
expire; unreachable objects can be pruned. Git explicitly notes that attaching
a Git note does not keep its target object alive. Aggressive pruning concurrent
with object creation can remove objects before their refs are installed.
[Garbage collection and retention](https://git-scm.com/docs/git-gc).

GitHub documents server garbage collection as part of removing sensitive data
after references have been removed. That procedure is not a storage guarantee
for arbitrary unreferenced objects, and it supplies no retention duration for
them. This research established no such hosted guarantee.
[GitHub removal procedure](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository#fully-removing-the-data-from-github).

**Design inference:** a content hash is an address, not a retention promise.
Accepted schema bytes and acceptance records can remain reachable through the
accepted commit history. An OID written inside a JSON string is not a native Git
edge that retains the named object. Temporary candidate trees may be discarded
after returning a check report unless a separate retention contract is chosen.
Do not promise durable candidate retrieval merely because a check returned an
OID. Protect accepted history from deletion or rewriting, and define deliberate
history-recovery behavior before treating it as the durable acceptance record.
Accepted-snapshot and release-copy retention follow ADR 0010; mechanisms for
preserving their availability remain separate from content identity.

## Source ancestry and authority are separate

`git merge-base --is-ancestor A B` tests ancestry and distinguishes a negative
answer from an operational error. This is available evidence for source
ordering, rather than an arrival-time or commit-time heuristic.
[Ancestry test](https://git-scm.com/docs/git-merge-base#Documentation/git-merge-base.txt---is-ancestor).

Commit signatures provide verifiable assertions about origin; GitHub describes
its verified status in terms of a cryptographically verifiable signature. That
is distinct from merely knowing an object ID.
[Signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification).

**Design inference:** record source repository identity with source revision;
authenticate the submission and enforce its permitted subgraph scope. Neither a
hash nor ancestry proves that submitted SDL was generated from that source,
that a source PR satisfied repository controls, or that deployed binaries expose
it. Missing history is unavailable evidence, not proof of divergent history.
The policy for older, divergent or rewritten source histories was subsequently
accepted in [ADR 0012](../adr/0012-order-publications-by-source-history.md).
Git supplies the ancestry relation; the application enforces the chosen policy.

## Keep distinct identities for distinct questions

The following factoring is a design proposal derived from the existing
contracts, not additional Git behavior:

| Identity | What it binds | What it should not accidentally mean |
| --- | --- | --- |
| Composition inputs | Exact schema/configuration tree and composer profile, including every consumed input | A source commit, request attempt, or approval |
| Directional check | Baseline and candidate composition identities, fixed baseline operation corpus, separately assessed corpus changes, and trusted check policy/tool identities | An undirected pair of trees or a continuously fresh result |
| Publication intent | Stable request key bound to the exact authorized subgraph replacements and source evidence | The registry head encountered during an internal retry |
| Registry acceptance | Commit that records the replacements and their acceptance record together | Artifact availability or gateway activation |
| Artifact bytes | Digest of the generated output, associated with its declared inputs | A claim that the inputs uniquely determined those bytes without checking |

A stable publication intent allows retry recovery after unrelated accepted
updates. Its original acceptance must be found in retained history before an
already completed request is reapplied. The acceptance record cannot contain
its own final commit ID without a circular identity; the containing commit
supplies that ID. Derived indexes may accelerate lookup but need not become a
second authority. Reusing a key for different input bytes must be an explicit
integrity failure. Accepted input identity and generated-byte identity stay
separate so nondeterministic or conflicting output is visible.

## Verification still required

The following remain untested: chosen credentials and repository rules,
GitHub conditional-write failure responses, protected-ref behavior, prepared
tree equivalence, response-loss reconciliation, retained-history lookup costs,
and hosted object retrieval/retention. No current source repository was supplied
to exercise ancestry or build provenance. Source ordering and semantic-failure
contracts are now accepted; their implementation and integration remain
unverified in [the design discussion](../design.md#current-decision-frontier).
