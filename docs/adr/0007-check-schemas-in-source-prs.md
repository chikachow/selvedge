# Assess generated schemas before source PRs merge

Schema changes arise from subgraph code changes, so their assessment must appear
on the subgraph's source PR before that PR merges. The first release therefore
includes source-PR integration, extending the earlier registry-PR-only framing.
The approval boundary is settled in
[ADR 0008](0008-source-prs-own-schema-approval.md); registry updates and supergraph
artifact publication remain distinct from gateway activation.
