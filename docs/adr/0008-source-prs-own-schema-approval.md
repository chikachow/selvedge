# Source PRs own schema approval

Source PRs are the proposal and approval surface for subgraph schema changes,
including compatibility exceptions governed by source-repository controls.
After source merge, trusted CI requests publication and the service records
the permitted schema/provenance update directly in the registry, avoiding a
second PR lifecycle. Registry maintainers retain policy, routing, and ownership
configuration; the service's registry-write authority is constrained to the
subgraphs its authenticated publisher may update.
