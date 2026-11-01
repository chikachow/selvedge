# Repository guidance

Preserve required behavior, public contracts, and security guarantees unless the
task authorizes changes. Design and testing methods are defaults; explain
consequential departures. Validation and review gates follow the policy below.

## Workflow

1. Inspect the worktree, affected code, tests, and relevant documentation. Identify
   the requested outcome, governing contracts, and overlapping changes before
   editing; preserve unrelated tracked and untracked work.
2. Resolve routine choices from repository evidence and carry prior agreement
   forward. Ask when unresolved requirements materially affect public contracts,
   compatibility, recovery, scope, or authorization; continue independent work
   while awaiting the answer. For reviews and plans, deliver analysis without
   unrequested implementation.
3. For implementation tasks, complete the requested outcome with affected code,
   tests, and owned documentation in agreement. Keep unrelated concerns separate;
   use another worktree when concurrent work needs isolation.
4. Follow the verification guidance below, then review the whole diff against the
   requested outcome and affected failure paths. For implementation tasks, resolve
   actionable in-scope findings and rerun affected checks. For reviews, report
   findings with their trigger, consequence, and evidence; distinguish defects,
   unmet outcomes, contract questions, and optional improvements.
5. Report each material requested outcome with evidence, checks, and limitations.
   Unavailable required checks or reviews leave verification incomplete: identify
   remaining work, the blocker, safe alternatives tried, and the input or access
   needed. Commit, push, publish, or deploy only within the user's authorization.
   Use Conventional Commits.

## Design and contracts

- Choose the simplest cohesive design for the whole affected concern, including
  callers and lifecycle transitions. Compare existing and proposed designs
  against the same requirements; distinguish required behavior, public contracts,
  and lifecycle guarantees from implementation choices. Weigh concrete benefits
  against migration, regression, review, and maintenance costs. Treat change size
  as a cost, not the objective.
- Preserve established domain, API, and protocol terminology. Rename internal
  symbols when a concrete correctness or comprehension benefit justifies the
  cost.
- Extract helpers when they expose a coherent operation or lifecycle boundary,
  remove meaningful duplication, or clarify the caller. Call count alone is not
  decisive. Testability can justify a production seam when the behavioral evidence
  warrants its interface and maintenance costs.
- Before adding defenses or removing safeguards, identify the failure scenario
  and required guarantees. Compare alternatives that preserve them; explain
  consequential or uncertain tradeoffs with evidence and explicit unknowns.
- For external input or trust-boundary changes, validate consumed values at their
  owning boundary and pass established guarantees inward. Preserve validated
  values while later decisions depend on their stability. Internal types alone
  do not establish trust. Keep secrets out of tracked files and output.
- Resolve material disagreements among the request, contracts, tests, and
  implementation before encoding behavior. Passing current tests alone does not
  establish the intended contract.

## Verification

- Keep verification proportionate to the change and satisfy existing repository
  gates. Waiving a validation or review gate requires explicit user agreement to
  that specific waiver; implementation authorization alone does not grant it.
- Implement checks enforcing agreed contracts autonomously. Add standing gates
  as contracts emerge, with maintainer agreement for additional obligations that
  can block future work, such as new approval requirements or coverage thresholds.
- Use the repository's pinned tools, scripts, and generators. For generated
  artifacts, edit authoritative inputs, regenerate outputs, and inspect tracked
  and untracked changes.
- For prose-only changes, check accuracy, local links, and the final diff; run
  `git diff --check` and any configured formatter. Check executable examples and
  contract changes for their behavioral effects. For instruction changes, walk
  through triggering, permitted, and unrelated tasks; label these as static
  walkthroughs. Claims of improved agent behavior require observed trials.
  Repeat or broaden passing checks only for new changes, failures, or unresolved
  concerns.
- Choose independent behavioral evidence at the affected boundary, including
  failure and recovery paths. Derive expectations from the governing contract
  rather than production logic. When consolidating tests, account for
  retained guarantees and justify consequential coverage removed. Claim regression
  detection only after observing unchanged expectations fail against faulty
  behavior and pass after the fix.
- Before claiming merge readiness, obtain independent review for consequential
  changes to who can exercise authority, credential exposure, or retention and
  recovery guarantees for accepted data or work. Judge the affected guarantees,
  regardless of filenames; incidental edits that leave those
  guarantees unchanged do not trigger this gate.
- An independent reviewer is a human or separate agent; the same model is
  acceptable. Give them the request, governing contracts, complete diff, relevant
  source, and tests without prescribing a verdict. They must assess that evidence
  themselves. If unavailable, continue other authorized work and report the
  missing review; self-review does not satisfy this gate.
- Support implementation claims with source or direct experiments and external
  compatibility claims with current primary sources or direct experiments.
  Distinguish static reasoning from execution, environmental failures from source
  defects, and local results from deployed behavior.

## Documentation

- Keep each fact and policy authoritative in one place. Put specialized guidance
  beside its concern, with explicit task-triggered reading pointers here. Keep
  evaluation history and completed plans with their change records. Repository
  guidance must work without personal agent configuration.
- Add setup and validation instructions when the corresponding tooling exists;
  keep commands and versions authoritative in its configuration.
- Add preventive rules for concrete, consequential failures, with a clear trigger
  and permitted path. Revise or remove rules that add friction without benefit.
