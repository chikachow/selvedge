# Selvedge

A self-hosted Federation schema registry design, backed by PostgreSQL. Selvedge
checks proposed graph configurations, retains accepted revisions and generated
artifacts, and supports explicit artifact selection by gateway deployments.

The repository currently contains design documentation, not an application.

- [System design](docs/design.md): scope, application contracts, guarantees,
  trade-offs and unanswered policy questions.
- [PostgreSQL data core](docs/postgresql-data-core.md): relations, invariants,
  concurrency, transactions, pgschema management, retention and verification limits.
- [Glossary](GLOSSARY.md): domain terminology.
- [Decision history](docs/decision-history.md): earlier alternatives and contract
  changes, with links to the preserved research and ADRs.
