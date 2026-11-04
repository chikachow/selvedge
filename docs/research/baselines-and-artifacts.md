# Baselines and artifact publication

Research date: 2026-11-01. These are findings and recommendations for the
[current design frontier](../design.md#current-decision-frontier), not accepted
publication or retention policy. Evidence is primary-source inspection and
static reasoning; no repository settings, publication workflows, or gateway
deployments were exercised.

Subsequent decision: [ADR 0010](../adr/0010-retain-validated-outputs-with-acceptance.md)
retains generated outputs with accepted registry inputs and uses releases as
distribution copies. The generation/storage alternatives below preserve the
earlier evaluation rather than define the current accepted workflow.

## Separate proposal impact from deployment impact

| Comparison | Explicit inputs | Question answered |
| --- | --- | --- |
| Candidate validity | Complete candidate graph and trusted tool configuration | Does this graph compose and pass the required validity checks? |
| Proposal impact | Exact target base and its combined merge candidate | What changes does this proposal introduce? |
| Deployment impact | Previously selected artifact and newly selected approved artifact | What changes between these deployment configurations? |

GitHub normally checks a synthetic merge result for `pull_request`. Strict
checks or a merge queue address target-branch changes after a check; a queue
also requires checks on `merge_group`.
[PR event semantics](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request),
[strict checks](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#require-status-checks-before-merging),
[merge queues](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue).

Inference: the baseline is the exact target revision used to construct the
candidate. An outdated PR head compared directly with the newest target can
misrepresent intervening additions as removals. The historical branch merge-base
does not represent the full current target either.

Production lag alone does not invalidate structural compatibility: preservation
of all previously valid documents is transitive. The important cases are earlier
accepted breaks and behavioral or routing changes that structural validity does
not cover. The following are constructed examples, not executed tests:

- Approved revision B removes `legacyName` using repository-authorized bypass.
  Revision D adds an unrelated field. B-to-D introduces no removal, but an
  upgrade from older artifact A to D still removes `legacyName`.
- B changes a default from 10 to 100. A-to-D can change behavior even if an
  operation omitting the argument remains valid.
- B moves a field with `@override`. A-to-D can change its resolving service
  while exposing the same public API.

The relevant semantics are documented by
[GraphQL field validation](https://spec.graphql.org/September2025/#sec-Field-Selections),
[argument coercion](https://spec.graphql.org/September2025/#sec-Coercing-Field-Arguments),
and [Apollo override](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/directives#override).

Recommendation: report the direct A-to-D delta rather than concatenate all
intermediate PR findings. A field added and removed entirely between deployments
never existed in A. Use retained artifact contents; recomposing old source with
a different tool does not necessarily reproduce the earlier selected artifact.

This needs no runtime control plane: deployment supplies explicit identities.
A previous deployment configuration is evidence of a previous selection, not
proof that every gateway currently runs it. The checker reports cumulative
impact without reinterpreting a previously accepted break as safe or inventing
an application approval step.

## Preserve operation evidence across proposal changes

Recommendation: use the baseline corpus unchanged against both baseline and
candidate schemas, then validate the proposed corpus separately and report its
changes. Otherwise, a PR removing a field and deleting its only known operation
could erase the operation regression from the report. This is a static failure
scenario; holding evidence fixed preserves the comparison while repository
controls can still accept a coordinated change. Missing historical coverage
cannot be replaced by newly supplied documents.

## Artifact storage alternatives

| Store | Benefit | Cost |
| --- | --- | --- |
| Generated files in source commits | Inputs and exact output share one Git snapshot; no separate publication writer if maintainers generate files | Regeneration and shared-output merge conflicts; trusted checks must reproduce bytes; source automation's permitted paths need a way to include shared generated outputs |
| Post-merge artifact commits in Git | Exact accepted-source generation; Git remains the storage interface | A writer, separate artifact history, ref protection, concurrent-write handling, and source/artifact identity mapping |
| Post-merge immutable GitHub releases | Exact accepted-source generation without generated-file PR churn or a second Git history | GitHub publication/download integration, publisher credentials, draft/upload/publish retries, and external retention obligations |

These trade-offs are design judgments. Source automation is currently authorized
only for its own schema and provenance, so proposing shared generated output
would require explicit handling rather than silently broadening that authority.

Git object identity depends on object contents; commits include their tree,
parents, and metadata. Reachable history is retained by normal garbage
collection, while unreachable objects can be pruned.
[Git data model](https://git-scm.com/docs/gitdatamodel),
[garbage collection](https://git-scm.com/docs/git-gc).

Consequences: a source commit cannot straightforwardly contain its own final SHA
inside an artifact manifest, because writing that SHA changes the commit. An
external artifact can name the accepted source SHA directly. A full commit SHA
establishes identity, not continued availability; retained refs and backups are
separate concerns.

GitHub immutable releases lock published assets and the associated tag, and
provide an attestation linking release assets to the tag and commit. The entire
release can still be deleted. Titles and notes remain editable, so immutable
provenance belongs in an attached metadata asset. The feature is opt-in and
affects future releases.
[Protection semantics](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases),
[enablement](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/establish-provenance-and-integrity/prevent-release-changes).

Ordinary Actions artifacts expire, with a 90-day default, and are deleted when
their workflow run is deleted. They are useful intermediate transport, not an
implicit permanent publication store.
[Artifact retention](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/remove-workflow-artifacts).

Recommendation: keep portable generated artifacts outside proposed source
commits, initially publishing them through a GitHub adapter to immutable
releases. The core still produces ordinary files. This is a proposal for the
next interview round; immutability is not a backup or retention guarantee.

## Publication acceptance and authority

The [accepted exception boundary](../adr/0001-repository-managed-compatibility-exceptions.md)
requires preserving compatibility failures after repository-authorized acceptance.
A publisher that unconditionally demands a compatibility pass would defeat
that boundary. Candidate validity and input/tool errors must remain distinct
from compatibility findings; exact result handling is awaiting a decision.

Recommended publication sequence:

1. Select the exact accepted source commit, graph directory, comparison baseline,
   and trusted tool/policy versions.
2. Revalidate and generate from those exact inputs; preserve the results rather
   than substituting the then-current branch head.
3. Publish the generated bytes with their digest, source and input identities,
   comparison baseline, tool/policy versions, and report.
4. Let deployment select the exact artifact. Bundling it into the deployment
   avoids depending on remote availability at each process startup.

GitHub's `contents: write` permission allows release creation. Publication
credentials belong to trusted publication execution, outside untrusted proposal
checks; merely receiving an artifact from another workflow does not establish
its trustworthiness.
[Workflow permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#permissions),
[workflow security](https://docs.github.com/en/actions/reference/security/secure-use).

An artifact digest or release attestation proves neither the adequacy of the
schema checks nor deployed-subgraph readiness. Approval authorization remains
with repository controls; the checker must not introduce its own approval system.
