# Finalize approved controls through the shared publisher

Toolchain and policy changes use separate maintainer PRs. Existing accepted
controls govern their admission; proposed controls are evaluated separately
against graph and operation inputs, including generated-output differences.
The same trusted finalizer used for schema publication applies the exact
approved control change to the latest graph, generates matching outputs, and
commits conditionally, preserving
[ADR 0010](0010-retain-validated-outputs-with-acceptance.md).

Maintainer review and approval remain repository responsibilities. The service
must bind the finalized change to the approved inputs without granting its own
approval or broadening source publishers' authority. This absorbs concurrent
schema updates and avoids accepting stale generated output, at the cost of a
dedicated control-PR finalization integration. The concrete GitHub review/commit
workflow and its enforcement remain to be selected and verified.
