# Repository-managed compatibility exceptions

Breaking public API changes remain visible as failing compatibility checks;
source-repository protection rules determine which users or groups may bypass
those checks and accept the change. Those rules and exception approvals are
outside Selvedge's implementation: it authenticates authorized publication
requests and preserves the compatibility result, without implementing reviewer
selection, bypass decisions, or a second registry exception-approval flow.
