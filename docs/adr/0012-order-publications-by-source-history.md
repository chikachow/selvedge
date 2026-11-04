# Order subgraph publications by source history

New publications advance through each subgraph's configured source history.
An overtaken source revision is superseded rather than allowed to restore older
SDL; intentional rollback uses a newer source revert. Record source progress
even when SDL bytes are unchanged, because artifact equality cannot establish
ordering. Divergent or rewritten history requires explicit resolution, and
missing ancestry evidence is not proof of divergence.

A coordinated batch keeps its submitted participant set. If a participant has
been superseded, the service does not silently discard it and publish a smaller
batch. The caller must choose a fresh complete publication; an already accepted
intent instead returns its original receipt under
[ADR 0011](0011-caller-owned-retries-and-publication-receipts.md). These rules
prevent accidental rollback while preserving independent concurrent updates
through [ADR 0009](0009-snapshot-checks-and-conditional-registry-updates.md).
