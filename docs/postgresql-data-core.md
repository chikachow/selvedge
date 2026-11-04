# PostgreSQL data core

Design specification, 2026-11-04. This document defines the relational model and
publication transaction for the [PostgreSQL-backed architecture](design.md).
The guarantees below are implementation requirements supported by isolated
experiments; no production schema or application is implemented.

## Guarantees and enforcement

| Guarantee | Enforcement |
| --- | --- |
| A committed registry revision contains its complete inputs, outputs and publication receipt | One transaction inserts the revision, publications and membership, then advances the graph's current revision |
| The current revision belongs to its graph; accepted history cannot branch or rewind through the publication API | Composite foreign keys, a unique predecessor constraint and a locked current-revision check |
| Each revision includes at most one publication of each subgraph | Membership primary key and composite foreign keys |
| Concurrent publication cannot overwrite a participant whose expected publication changed | Per-subgraph preconditions checked at acceptance |
| A batch is accepted in full or has no effect | All participants share one graph transaction; no partial retry or participant omission |
| Repeating an accepted request returns its original result without changing current state | Graph-scoped unique idempotency key, immutable request binding and receipt lookup before freshness checks |
| Accepted history, including old memberships and artifact bytes, is immutable to ordinary application access and has no automatic expiry | Runtime access is limited to reads and explicitly granted operations; direct table writes are denied |
| An empty graph remains a valid registry state | A real current revision with zero membership rows and null composed artifacts |

These guarantees assume a trusted application supplies the exact evaluated inputs
and outputs. SQL does not establish Federation validity, authorization or artifact
derivation. Database administrators remain trusted, and crash durability and
recovery depend on the deployment configuration.

## Relational model

| Relation | Contents |
| --- | --- |
| `graphs` | Graph identity and `current_revision_id` |
| `registry_revisions` | Revision UUID, predecessor, idempotency key, bound request, graph-level inputs, exact generated artifacts, checker identity, report and publication metadata |
| `subgraph_publications` | Subgraph name, accepting registry revision, exact SDL, source provenance and artifact-defining configuration such as routing |
| `revision_subgraphs` | The exact subgraph publications included in each registry revision |

A subgraph publication is identified by
`(graph_id, subgraph_name, accepted_revision_id)`. Its accepting registry revision
is also the version used by optimistic concurrency. No independent version
counter or content-addressing subsystem is required. Membership references avoid
copying unchanged SDL and give foreign keys authoritative input records to check.

A source revision identifies source code; an artifact digest identifies bytes.
Neither identifies publication order. Every newly accepted subgraph publication
receives a new identity even if its SDL is unchanged. Otherwise an intervening
publication or an A-to-B-to-A content change could satisfy an obsolete condition.
An accepted-request retry creates neither a revision nor a publication.
See the [glossary](../GLOSSARY.md) for domain terminology.

### Keys and constraints

The following is a constraint sketch, not an executable migration:

```text
graphs
  PRIMARY KEY (graph_id)
  current_revision_id NOT NULL
  FOREIGN KEY (graph_id, current_revision_id)
    -> registry_revisions (graph_id, revision_id)
    DEFERRABLE INITIALLY DEFERRED

registry_revisions
  PRIMARY KEY (graph_id, revision_id)
  FOREIGN KEY (graph_id) -> graphs (graph_id)
  FOREIGN KEY (graph_id, previous_revision_id)
    -> registry_revisions (graph_id, revision_id)
  UNIQUE (graph_id, idempotency_key)
  UNIQUE NULLS NOT DISTINCT (graph_id, previous_revision_id)

subgraph_publications
  PRIMARY KEY (graph_id, subgraph_name, accepted_revision_id)
  FOREIGN KEY (graph_id, accepted_revision_id)
    -> registry_revisions (graph_id, revision_id)

revision_subgraphs
  PRIMARY KEY (graph_id, revision_id, subgraph_name)
  FOREIGN KEY (graph_id, revision_id)
    -> registry_revisions (graph_id, revision_id)
  FOREIGN KEY (graph_id, subgraph_name, publication_revision_id)
    -> subgraph_publications
       (graph_id, subgraph_name, accepted_revision_id)
```

Identity and request-binding fields are non-null. Only the initial revision has
a null predecessor, as enforced by graph creation and publication. Composite
foreign keys prevent references across graphs or to another subgraph's
publication. The unique predecessor constraint permits at most one initial
revision and one successor per revision; it does not establish membership
completeness or replace the publication function's current-revision check.

The deferred current-revision reference permits graph creation and its initial
revision in one transaction. The column remains non-null, so creation cannot
commit a graph without a revision. Ordinary publication requires an existing
graph; bootstrap is a separate authorized operation.

A row-level `CHECK` requires the supergraph and public API schema artifacts to be
null together. The creation/publication operation additionally requires both to
be absent exactly when membership is empty. Cross-row invariants belong in
relational constraints or the restricted operation, not queries hidden inside
`CHECK` functions. `NULLS NOT DISTINCT` requires PostgreSQL 15 or later; the
deployment version remains to be confirmed.
[PostgreSQL constraints](https://www.postgresql.org/docs/18/ddl-constraints.html).

### Stored bytes

Use `bytea` for exact SDL and generated artifacts. Compute digests from the
stored bytes rather than trusting independently supplied hashes. Use JSONB for
structured diagnostics and flexible provenance/configuration, but do not
reconstruct byte-preserved inputs from JSONB. Graph membership and publication
identity remain relational.
[Binary data](https://www.postgresql.org/docs/18/datatype-binary.html),
[binary hashes](https://www.postgresql.org/docs/18/functions-binarystring.html),
[JSON storage](https://www.postgresql.org/docs/18/datatype-json.html).

## Optimistic concurrency

Caller conditions apply to subgraphs, not the entire graph:

| Requested operation | Condition at acceptance |
| --- | --- |
| Add | The subgraph name is absent from the current revision |
| Replace | The current publication equals the caller's expected publication |
| Remove | The current publication equals the caller's expected publication |

A batch has one condition per participant; all must hold. Preserve these
conditions throughout retries. In particular, refreshing an expected publication
to make a stale request succeed would authorize an overwrite the caller did not
request. A new removal of an already absent subgraph conflicts; a retry of an
accepted removal recovers its original receipt.

The **evaluated registry revision** serves a separate purpose: it identifies the
complete graph used for checking. It must still be current when the transaction
accepts the result. An unrelated publication invalidates that evaluation, so the
service reloads, recomposes and retries with the original subgraph conditions.
The evaluated revision may change during retry and is excluded from request
binding; the caller's conditions remain fixed and bound.

For example, Products' publication was accepted in R12. Reviews advances the
graph to R13 without changing Products. An update expecting Products at R12 can
be recomposed against R13 and accepted. If Products has advanced, that request
conflicts even if the new SDL matches the old bytes. R12/R13 are illustrative
labels, not an additional sequence-number scheme.

**Create-if-absent trade-off:** a never-accepted delayed addition may succeed
after another request creates and removes the same name, provided it remains
authorized. This is intentional: the condition describes current absence, not
absence throughout an interval. Preventing that recreation would require
retained deletion versions, history inspection or a graph-wide precondition
that also rejects unrelated changes. Those mechanisms are excluded.

### HTTP encoding

The proposed publication API uses explicit per-change conditions for both single
updates and batches: replacements/removals carry `expected_revision`; additions
require absence. A failed participant condition returns `409 Conflict`. Internal
graph-revision conflicts trigger bounded reconstruction instead. Successful
checking neither reserves state nor grants publication authority.

`ETag`/`If-Match` is appropriate for direct mutations of an individual subgraph
resource. A strong ETag must identify its complete representation, including
variants, rather than merely hash SDL. However, an `If-Match` list expresses
alternatives for one target, not simultaneous conditions on several subgraphs.
`If-None-Match: *` on a publication collection likewise cannot express absence
of a particular subgraph. Explicit per-change conditions avoid extra endpoints
or a custom revision header solely to represent a batch.
[HTTP validators](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.8),
[conditional requests](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1).

This interface also separates receipt recovery from HTTP conditional creation:
failed `If-None-Match` on an unsafe method requires `412`, whereas `If-Match`
permits recognizing an already successful request. Recovered receipts identify
the historical accepted revision; they must not present its ETag as the current
subgraph's validator.
[If-Match](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1.1),
[If-None-Match](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1.2).

## Publication transaction

The application authenticates the caller, validates the request binding and
checks for an existing accepted receipt **before recomposition**. A matching
receipt is returned subject to result-access authorization; replay does not
require the old request to pass today's checker. A missing receipt is not proof
that another attempt cannot still commit.

For a new attempt, load an immutable registry revision, assemble only authorized
changes and run the checker outside the write transaction. Reading its identity
and then its immutable rows does not require a long read transaction. Supply the
evaluated revision, bound request and exact generated result to one restricted
publication function.

At `READ COMMITTED`, the function:

1. Locks the graph row with `SELECT ... FOR UPDATE`.
2. In a separate statement after acquiring the lock, repeats the idempotency
   lookup. A matching accepted request returns its original revision; changed
   request contents fail. This handles acceptance racing with the application
   lookup or evaluation.
3. Checks that the current revision is the evaluated revision and verifies every
   participant's original condition.
4. Allocates a fresh revision UUID and inserts the complete result, new subgraph
   publications and derived membership. Untouched publications are preserved;
   removal omits membership only from the new revision.
5. Advances `graphs.current_revision_id`. All writes commit together.

An uncaught error rolls back the operation; failed conditions produce no partial
revision or batch. The returned revision ID is provisional until the enclosing
transaction commits. Report acceptance only after commit succeeds, or recover it
through the accepted-request lookup if the commit outcome is uncertain.

The graph lock covers only the database operation. Different graphs use different
locks; composition and external calls never hold one. A modifying `VOLATILE`
function at `READ COMMITTED` obtains fresh snapshots for successive statements,
allowing the post-lock lookup to observe the preceding publisher's commit.
[Row locks](https://www.postgresql.org/docs/18/explicit-locking.html),
[isolation](https://www.postgresql.org/docs/18/transaction-iso.html),
[function volatility](https://www.postgresql.org/docs/18/xfunc-volatility.html).

`SERIALIZABLE` cannot prove that earlier, external composition used the current
graph; the evaluated-revision check remains necessary. A conditional `UPDATE`
could implement acceptance, but zero affected rows is not an SQL error and would
require explicit rollback of preceding inserts. The row-lock sequence checks
eligibility before mutation and makes that failure path explicit.
[UPDATE](https://www.postgresql.org/docs/18/sql-update.html).

## Idempotency and access control

Bind each graph-scoped idempotency key to the stable authenticated publishing
identity, exact requested changes and source claims, and original subgraph
conditions. The identity must survive legitimate authentication-token renewal
and transport retries. Its precise mapping is part of the request contract.
Do not bind the changing evaluated revision or generated outputs.

Compare the retained normalized request, or compute its fingerprint inside the
trusted operation. Never accept an unchecked caller hash as proof of equality.
The function derives inserted subgraph inputs from the same bound request, not
an independently supplied change list. Possession of a key grants neither
publication authority nor access to its receipt.

After uncertain commit, retry with the same key and request. The graph lock and
repeated lookup serialize competing attempts and recover an accepted result even
when later revisions are current. An unreachable writer leaves the outcome
unknown; absence on a lagging replica does not establish non-acceptance. Rejected
requests create no durable receipt or reserved key. There is no background
completion promise.

The runtime role has read access and narrowly granted function execution, but no
direct `INSERT`, `UPDATE`, `DELETE` or `TRUNCATE` privileges. Denying only updates
and deletes would still permit insertion into an already accepted revision.
Use a non-login owning role and a `SECURITY DEFINER` function with schema-qualified
references, a trusted `search_path`, revoked default `PUBLIC` execution and
explicit grants. The runtime role must not own or inherit ownership of the tables.
These controls protect against ordinary application mutations, not trusted owner
or administrator actions.
[Function security](https://www.postgresql.org/docs/18/sql-createfunction.html#SQL-CREATEFUNCTION-SECURITY),
[privileges](https://www.postgresql.org/docs/18/ddl-priv.html).

## Schema management

Use [pgschema](https://www.pgschema.com/) for declarative schema management.
Version-controlled SQL describes the desired schema; pgschema derives an ordered
DDL plan against the target. Generate and inspect a target-specific saved plan,
then apply it with the same pinned tool version and fingerprint validation enabled.
The desired SQL is authoritative; generated plans are deployment artifacts, not
a separately maintained sequence of handwritten schema migrations.
[Planning](https://www.pgschema.com/cli/plan),
[applying saved plans](https://www.pgschema.com/cli/apply).

The tooling boundary must preserve the ownership and privilege model above:

| Managed by pgschema | Established outside pgschema |
| --- | --- |
| Supported tables, constraints, functions and explicit object grants | Database/schema creation, roles and membership, object ownership and global default privileges |

Unsupported ownership and global-default statements are ignored with warnings;
they must not be mistaken for enforced desired state. New objects belong to the
effective applying role. Migration access is separate from runtime access and
must establish the designated non-login owner without granting ownership to the
runtime role. The concrete connection/role-switching mechanism needs validation.
[Supported scope and exclusions](https://www.pgschema.com/syntax/unsupported).

Before creating functions, trusted bootstrap must revoke the global default
`EXECUTE` grant to `PUBLIC` for the effective object-creating role. Membership in
the designated owner role alone does not apply its defaults; per-schema revocation
cannot remove a global default grant. Explicit function ACLs remain managed. Verify
ownership, `SECURITY DEFINER`, trusted `search_path`, execution grants and denied
runtime table writes after applying changes; pgschema's schema fingerprint does
not cover its unmanaged administrative state.
[PostgreSQL default privileges](https://www.postgresql.org/docs/18/sql-alterdefaultprivileges.html).

A schema apply is **not globally atomic**: pgschema uses transaction groups, and
operations such as concurrent indexes run outside a transaction. A later failure
can leave earlier groups committed. Schema rollout therefore needs explicit
handling of partial application, DDL locks, data backfills and compatibility with
active application versions. Publication's atomicity guarantee does not extend
to schema deployment. A generated diff alone does not establish safe data
transformation or preserve accepted history.
[Transaction handling](https://www.pgschema.com/cli/apply).

Renames require explicit treatment: pgschema does not infer them or manage data
state. Inspect drop/recreate operations and sequence necessary backfills;
reverting desired SQL cannot recover deleted accepted history.
[Data migrations](https://github.com/pgplex/pgschema/blob/v1.13.4/docs/workflow/data-migration.mdx),
[rollback](https://www.pgschema.com/workflow/rollback).

The compatibility review inspected pgschema v1.13.4 source and examples; it did
not execute this schema through pgschema. Upstream fixtures cover
[function security settings](https://github.com/pgplex/pgschema/blob/v1.13.4/testdata/diff/create_function/issue_335_search_path_rewrite/plan.sql)
and [null-aware uniqueness](https://github.com/pgplex/pgschema/blob/v1.13.4/testdata/diff/create_table/add_unique_constraint_nulls_not_distinct/plan.sql).
Before implementation, pin the tool and PostgreSQL versions and verify fresh
installation, upgrade convergence, privilege bootstrap and partial-failure
recovery on a representative database. Verify cyclic foreign-key creation order:
deferred row checks do not defer DDL name resolution. No schema files or migration
pipeline exist yet.

## Empty graphs

Creation may commit an initial revision with zero membership rows. Removing the
last subgraph similarly creates an empty revision while retaining graph identity,
non-null current revision, historical memberships, inputs and artifacts. Empty
and nonexistent graphs are distinct states.

An empty revision stores SQL `NULL` for both composed artifacts. It must not use
zero-byte SDL, borrow a historical artifact or introduce a synthetic service.
Input validation and authorization still apply. Composition and comparisons
requiring an absent API schema are reported as unavailable, never as successful
checks; the membership changes remain visible in the report.

**Compatibility boundary:** returning from an empty to a nonempty revision has
no immediate-baseline API schema to compare. There is no implicit last-nonempty
baseline. Deployment comparison can still use explicitly selected historical
artifacts; whether release policy must require that comparison is an open system
decision in the [architecture](design.md).

An empty revision cannot be selected as an executable gateway artifact. Earlier
artifacts remain retrievable, and an already configured gateway retains its
explicitly selected artifact. Registry removal does not stop that gateway or
remove its runtime dependency on deployed subgraphs. Adding subgraphs resumes
ordinary composition and artifact production.

## Trade-offs and unresolved details

| Choice | Benefit | Cost or limit |
| --- | --- | --- |
| Relational immutable snapshots | Foreign keys validate membership references; unchanged SDL is reused | The publication operation must construct complete membership; rows grow with accepted revisions |
| Short graph-row lock | Simple atomic acceptance and receipt recovery without long-running composition locks | Final writes serialize per graph; contention may require repeated composition |
| Per-subgraph conditions | Unrelated publication can retry automatically | The complete graph must still be reevaluated; retries need a bound |
| Create-if-absent | No tombstones or historical absence protocol | A never-accepted delayed addition can recreate a removed name |
| Inputs and outputs in one database transaction | Acceptance and artifact availability cannot diverge through a second storage write | Database storage and backups include generated artifacts; no automatic pruning |
| Restricted database function | One complete mutation interface protects accepted history | Bootstrap and schema migrations require separate trusted operations; administrators remain trusted |

Before implementation, specify:

- PostgreSQL deployment version, write durability, failover behavior, backup and
  restore procedures, acceptable recovery point and handling of lost accepted
  records. Restoring an older database can erase acknowledged receipts;
  idempotency alone cannot reconstruct them or authorize automatic replay.
- The complete request schema, stable publisher identity, canonical equality
  rules, binary encoding and mapping between API conditions and stored inputs.
- Input/artifact size limits, transaction and checker timeouts, contention retry
  limits and the API response when an attempt exhausts its budget.
- Authorized graph creation and the pgschema/bootstrap deployment procedures
  above, preserving the same history and reference invariants.

Authentication, source-release selection, configuration lifetime and checker
rollout remain application concerns described in the [architecture](design.md).
The database does not prove artifact derivation, Git ancestry, immediate
revocation or runtime compatibility. WAL and storage configuration also determine
which acknowledged commits survive failures.
[WAL configuration](https://www.postgresql.org/docs/18/runtime-config-wal.html).

## Verification evidence

Isolated local PostgreSQL 18.6 fixtures exercised the four-relation model and
restricted function. Independent review assessed the design and experiment
results. The fixtures had no network access or published ports and used temporary
storage. Observations included:

- Concurrent requests against one evaluated revision accepted at most one new
  result; matching requests recovered one acceptance. Unchanged additions retried
  after unrelated advancement, while competing same-name additions conflicted.
- Changed request bindings and stale participant conditions failed. Accepted
  replay after later revisions returned the original result without rewinding
  current state; replay after removal did not recreate membership.
- Explicit rollback, stale batch participants and errors after inserting the
  first participant left no partial revision, publication or membership.
- Runtime history mutation, historical membership insertion, pointer rewind and
  truncation were denied. Cross-graph references, duplicate initial revisions
  and duplicate successors violated constraints.
- Binary outputs and UTF-8 SDL with CRLF survived exact byte round trips. Separate
  UUID/non-null/deferred-reference creation checks rejected incomplete graphs.
- Empty → nonempty → empty → recreated lifecycles preserved historical artifacts.
  Null artifacts serialized to JSON `null`. A never-accepted addition succeeded
  after intervening creation and removal, confirming the accepted absence rule.

The main fixture used integer revision labels and nullable bootstrap setup;
separate fixtures checked UUID and non-null creation. Request binding used a
fixture JSONB shape, not the final API. Composition outputs were fixture bytes;
recomposition was represented by resubmission against a new evaluated revision.
Actual response loss, HTTP, authentication, crash durability, failover and
backup/restore were not exercised. These results establish database-operation
evidence, not end-to-end or deployed guarantees.

Separate experiments with `@apollo/composition@2.14.4`, `@apollo/gateway@2.14.4`
and `graphql@16.14.2` rejected zero-subgraph composition with `NO_QUERIES`, empty
Gateway service lists and empty SDL. GraphQL requires a query root and at least
one field per object, so an empty `Query` is not a valid workaround. These results
support absent artifacts; they do not verify the eventual Rust integration or
the deployed gateway version.
[Apollo composition errors](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/errors),
[GraphQL root types](https://spec.graphql.org/September2025/#sec-Root-Operation-Types),
[object validation](https://spec.graphql.org/September2025/#sec-Objects.Type-Validation).
