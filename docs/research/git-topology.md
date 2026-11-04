# Git topology for graph configurations

Research date: 2026-11-01. Recommendation for discussion, not an accepted
topology or implementation specification. This note follows the decisions to
keep [compatibility exceptions in repository controls](../adr/0001-repository-managed-compatibility-exceptions.md)
and [deployment environments outside the domain model](../adr/0002-no-environment-domain-model.md).

## Recommendation

Make one complete graph configuration directory the core input boundary. The
checker receives explicit baseline and candidate directories; their parent
directories, repository names, and branch names have no domain meaning. Use
**one graph configuration per repository, at its root, as the initial operating
convention**. Do not make that convention a core restriction.

This is a design judgment, not a measured cost result. It aligns each graph's
history, required checks, compatibility bypass groups, and merge queue with
GitHub's existing repository and branch boundaries. It avoids initially needing
multi-graph discovery, changed-path dispatch, or per-graph check naming. It does
not assume how many graphs or deployments exist. One configuration can be used
by several deployments; Selvedge need not infer their relationship.

Top-level graph directories in one repository remain a coherent alternative
when the operator wants shared administration and coordinated Git changes. That
arrangement needs no environment or variant model either. Choose it when those
benefits justify the shared authority and CI dispatch described below. Moving a
configuration into or out of a repository should change integration wiring,
not GraphQL validation semantics.

## Compare the same requirements

The comparison assumes complete federation validation, concurrent-proposal
checking, source automation limited to its owned subgraph SDL and provenance,
and maintainer control of policy and routing. No option by itself enforces all
of those requirements.

| Arrangement | Concrete benefit | Cost and shared failure boundary |
| --- | --- | --- |
| One graph configuration per repository | Repository-scoped credentials, bypass groups, check names, and queues naturally match the configuration being checked. All subgraphs of that graph can still change in one PR. | Repository and integration settings repeat. Changes across graph configurations require separate PRs and cannot become one Git commit. Shared Apps or administrative access can still span repositories. |
| Several complete configurations in top-level directories of one repository | One PR can record coordinated changes across graphs; common administration and integration are maintained together. The core can check each directory independently. | Credentials and repository administration span those directories. Shared branch checks and queue require deliberate dispatch; directory ownership does not create independent credentials. A mistaken shared workflow or ruleset can affect every graph. |
| One configuration per long-lived branch in one repository | Branch-specific protection rules and CODEOWNERS can distinguish accepted histories without adding an environment model. | Credentials remain repository-scoped. Configuration and integration changes must be propagated across histories; one PR cannot target several branches. This adds branch-management work without the access separation of separate repositories. |

These costs are qualitative inferences from the GitHub boundaries below.
Cross-graph changes in one commit provide atomic Git recording, not atomic
service deployment. Separate repositories do not prevent coordinated changes to
multiple subgraphs within one graph.

## What GitHub actually scopes

### Compatibility bypass is a ruleset boundary

GitHub grants bypass to roles, teams, or Apps for a ruleset. It also offers
pull-request-only bypass. Branch rulesets select branches or tags; those
selectors do not select graph directories. [Creating rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository).

Multiple rulesets can apply together, and applicable rules aggregate. Therefore
the user-selected exception mechanism can put the breaking-change check in its
own ruleset, with the designated groups on that ruleset's bypass list. Keep
composition, provenance, ownership, and other non-waivable requirements in
separate applicable rulesets without that compatibility bypass. This separation
is a design inference from GitHub's ruleset granularity; a bypass is not a
per-diagnostic approval. [Rule layering](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets#about-rule-layering).

In a shared repository, different exception groups for different graphs require
distinct graph check names and corresponding rulesets. One aggregate
compatibility check with one bypass list grants the same exception authority
across every graph that check covers. A PR affecting several graphs still has
to satisfy or legitimately bypass each applicable requirement. No Selvedge
approval database or exception state machine is needed.

Required status checks can be bound to an expected GitHub App. Strict checks
also require an up-to-date branch; loose checks permit merging without that
condition. The chosen check producer and its workflow still need to be trusted:
an App identity alone does not establish the integrity of the code it runs.
[Required status checks](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-status-checks-to-pass-before-merging).

### Path ownership is not credential isolation

CODEOWNERS assigns reviewers to paths, using the PR's base-branch file. Owners
must have repository write permission. When owner approval is required, either
owner on the same matching entry can satisfy it; listing source owners and
registry maintainers together does not require both. The last matching pattern
wins, and the ownership file itself needs protection. [CODEOWNERS behavior](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners).

GitHub App installation tokens can be narrowed to selected repositories and
permissions. The documented token-scoping parameters do not include directory
paths. Thus a path convention or CODEOWNERS entry cannot make a repository write
token an owned-subgraph-only credential. [Installation token scope](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app).

GitHub also supports push rulesets that restrict changes to file paths. Those
are additional push controls with ruleset bypass, not directory-scoped token
permissions. Their availability and applicability would need checking before
relying on them. [File-path restrictions](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#restrict-file-paths).

Consequence for the agreed authority contract: even a one-graph repository
contains several subgraphs and maintainer-owned configuration. Topology alone
does not stop source automation from proposing changes outside its authority.
A trusted submission or validation boundary must enforce the actor-to-subgraph
mapping and reject unauthorized path changes. Its exact mechanism remains open;
this note does not add one. Separate repositories reduce credential scope only
when the actual credentials and administration are separately scoped too.

### Concurrency follows the target branch

A merge queue tests a candidate against the latest target branch plus earlier
queued changes. GitHub Actions must handle `merge_group` for queue checks.
Build concurrency is configurable, so a shared queue does not imply that all
builds execute serially. [Merge queue behavior and CI requirements](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue).

Inference: unrelated graph directories on one protected branch still share its
integration sequence and required-check reporting. Separate repositories allow
independent queues, but every graph still needs validation of the exact combined
candidate that will be accepted. Checking each original PR once against an old
baseline does not establish that concurrent changes compose together.

For a multi-directory repository, a required workflow skipped by path filters
leaves its check pending and can block a PR. A conditional skipped job instead
reports success. Therefore the integration must deliberately report every
required graph check and distinguish an unaffected graph from an untested one;
shared input changes may affect several graphs. [Required-check filtering behavior](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks).

## Facts still needed before configuring GitHub

- Whether graph configurations actually need different source credentials,
  maintainers, or compatibility bypass groups. Shared deployment ownership alone
  does not answer this.
- Whether coordinated changes across graph configurations are common enough to
  justify shared history and CI dispatch. No volume or cost measurements exist.
- The hosting plan, repository visibility, active organization rulesets,
  administrative bypass rights, and check-producing App. Documentation does not
  prove the operator's actual controls. In particular, private-repository merge
  queues require an organization using GitHub Enterprise Cloud. [Merge queue availability](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue#who-can-use-this-feature).
- How source automation's proposal authority will be enforced within a graph,
  independently of the selected repository layout.

Verification: current official GitHub documentation was read; no repository
settings, bypass behavior, credentials, or merge queues were exercised. The
recommendation is deliberately separate from an assertion that any existing
repository is configured safely.
