# Publish immutable artifacts after registry acceptance

Status: superseded by [ADR 0010](0010-retain-validated-outputs-with-acceptance.md).
The original decision below is retained as history; ADR 0010 carries forward
the retention policy while replacing generation and storage mechanics.

After an accepted registry update, trusted CI generates the supergraph and
provenance from that exact registry revision and initially publishes them as
immutable GitHub release assets. This keeps generated output out of source PRs
and avoids a separate artifact Git history; GitHub publication belongs to the
adapter while the core produces ordinary files. Published artifacts are retained
without automatic expiry or pruning; deliberate operator deletion remains
outside Selvedge.
