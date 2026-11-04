# Selvedge

A self-hosted Federation schema registry design, backed by PostgreSQL. Selvedge
checks proposed graph configurations, retains accepted revisions and generated
artifacts, and supports explicit artifact selection by gateway deployments.

The repository currently contains design documentation, not an application.

- [System design](docs/design.md): scope, application contracts, guarantees,
  trade-offs and unanswered policy questions.
- [PostgreSQL data core](docs/postgresql-data-core.md): relations, invariants,
  concurrency, transactions, request binding and access control.
- [Operations](docs/operations.md): bundled gateway artifacts, writer/control
  handover, pgschema deployment and recovery boundaries.
- [Implementation guidance](docs/implementation.md): prerequisites, delivery
  sequence, acceptance scenarios and existing evidence.
- [Glossary](GLOSSARY.md): domain terminology.
- [Decision history](docs/decision-history.md): earlier alternatives and contract
  changes, with links to the preserved research and ADRs.

Start with the system design. Consult the data core when implementing persistence,
operations when changing deployment or recovery, and implementation guidance when
planning or verifying delivery. Open policy decisions block the affected contract;
they are not implicit permission to weaken earlier guarantees.
