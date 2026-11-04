# PostgreSQL data core

Status: proposed data model and transaction interface, 2026-11-04. The user wants
to define application consistency through PostgreSQL while keeping the solution
simple and using established terminology. This develops the
[current architecture](design.md); it is not a production migration or an
implementation of the GraphQL checker.

## Domain and consistency requirements

A **registry revision** records one accepted graph configuration and its generated
artifacts. A **subgraph publication** records an accepted update to one subgraph.
A **source revision** identifies source code; it does not identify generated SDL
or a database revision. An **idempotency key** identifies a caller's request across
retries. A digest identifies bytes, not the order of publications. These meanings
follow the [glossary](../GLOSSARY.md) and standard database terminology.

The data core must preserve these properties:

1. Every visible registry revision has its complete inputs and outputs. Zero
   subgraphs is valid registry state; that revision has no composed artifacts.
2. The current registry revision belongs to the graph and advances only through
   publication. Gateway rollback selects a historical artifact without rewinding
   registry history.
3. Each revision includes a subgraph at most once and refers to its exact retained
   publication. Unchanged subgraphs keep their existing publication references.
4. Publishing a batch changes all its participants together. An intervening change
   to a participant cannot be silently overwritten.
5. A repeated accepted request returns its original revision. Reusing its
   idempotency key for different requested changes is an error.
6. Ordinary application access cannot modify accepted history, including adding
   members to an existing revision.

These requirements justify relational keys, foreign keys, restricted privileges
and one publication function. They do not require a queue, event-sourcing system,
separate receipt ledger or arbitrary executable checks inside the database.

## Four relations

| Relation | Purpose and principal fields |
| --- | --- |
| `graphs` | Graph identity and `current_revision_id` |
| `registry_revisions` | Revision identity, previous registry revision, idempotency key and request binding, complete generated artifacts, checker identity, retained graph-level inputs, report and publication metadata |
| `subgraph_publications` | Subgraph name, accepting registry revision, exact SDL, source provenance and artifact-defining subgraph configuration such as routing |
| `revision_subgraphs` | The subgraph publications included in each registry revision |

Use UUIDs for registry revision identities. A subgraph publication is identified
by `(graph_id, subgraph_name, accepted_revision_id)`; no extra version counter or
schema-content identifier is needed. Its accepting registry revision supplies
the version used by an optimistic-concurrency precondition.

For example:

| Registry revision | Products publication | Reviews publication |
| --- | --- | --- |
| R12 | accepted in R12 | accepted in R9 |
| R13, updating Reviews | accepted in R12 | accepted in R13 |

R12/R13 are explanatory labels, not an additional numeric versioning scheme.
A new Products publication advances its reference even if its SDL is unchanged.
Otherwise, intervening source changes or an A-to-B-to-A schema change could make
an old precondition appear current. A retry of the same accepted request creates
neither a new registry revision nor a new subgraph publication.

The fourth relation separates the identity of an accepted input from the graph
revisions that use it. This gives a foreign key one authoritative input record
to reference and avoids copying unchanged SDL. It is not a general blob store
or content-addressing subsystem.

## Optimistic concurrency

The user selected per-subgraph conditions with automatic retry across unrelated
publications. A replacement or removal identifies the subgraph publication the
caller observed. The service preserves that expected revision throughout retries;
it must not refresh it to overwrite an intervening publication. A batch carries
one condition per participant, and all must hold for the batch to be accepted.

For additions, the agreed behavior is ordinary create-if-absent: the named
subgraph must be absent when the request is accepted. This allows recomposition
after unrelated changes without adding another relation or a graph-wide caller
precondition. It does permit a never-accepted delayed addition after another
request creates and removes that name, provided the delayed request is still
authorized. This consequence is accepted. Preventing that recreation would
require retained deletion versions, history inspection, or a fixed graph revision
that also conflicts with unrelated changes. Those mechanisms are not part of
the selected contract.

Two revisions have different jobs:

- The caller's **expected subgraph publication** prevents overwriting that
  participant. It belongs to the immutable request and idempotency binding.
- The service's **evaluated registry revision** identifies the complete graph
  used for composition. PostgreSQL must compare it with the current revision
  before accepting the result. It can change during recomposition and is not
  part of the caller's immutable request.

For example, an update to Products expects its publication accepted in R12.
Reviews advances the graph to R13 while Products remains at that publication.
The service can recompose against R13 and retry with Products still expecting
R12. If Products itself has advanced, the original request conflicts even when
the new SDL happens to match the old bytes.

### HTTP representation

Prefer one publication request shape for both single changes and batches:
each replacement/removal carries its `expected_revision`, while an explicit
addition requires absence. An unconditional upsert would erase this distinction.
Use an application-level `409 Conflict` when a participant's condition fails;
an internal graph-revision mismatch instead triggers bounded reconstruction and
rechecking. A successful check does not reserve the graph or authorize a write.

Standard `ETag`/`If-Match` is appropriate if reads and mutations target the same
individual subgraph resource. A strong ETag can include its publication identity
and representation version; it must cover all represented data and variants,
not just SDL. The service must not include unrelated changing graph data in that
representation while promising an unchanged strong ETag. A failed `If-Match`
produces `412 Precondition Failed`; a required but missing condition can produce
`428 Precondition Required`.
[HTTP validators and conditions](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.8),
[428](https://www.rfc-editor.org/rfc/rfc6585.html#section-3).

However, an `If-Match` list offers alternatives for one target, not conditions
on multiple subgraphs. Nor can `If-None-Match: *` on a publication collection
mean that a particular subgraph is absent. The shared publication endpoint
therefore uses explicit per-change conditions rather than a custom revision
header or extra single-subgraph endpoints just to use conditional headers.
An expected revision is still ordinary optimistic concurrency; its transport
does not change the database check.
[Conditional requests](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1).

This also keeps accepted-request recovery consistent: matching idempotency keys
return the original receipt without performing another mutation. Raw HTTP
conditional creation needs separate care because failed `If-None-Match` on an
unsafe method requires `412`, whereas `If-Match` explicitly allows recognizing
an already successful request. Do not silently treat them as the same rule.
Recovered results identify their historical accepted revision; they must not
advertise its ETag as the current subgraph's validator.
[If-Match](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1.1),
[If-None-Match](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1.2).

## Declarative constraints

The essential keys and references are:

```text
graphs
  PRIMARY KEY (graph_id)
  current_revision_id NOT NULL
  FOREIGN KEY (graph_id, current_revision_id)
    -> registry_revisions (graph_id, revision_id)

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

Required identity and request-binding fields are `NOT NULL`. The supergraph and
public API schema artifacts are both null for an empty revision and both present
for a nonempty revision. A row-level check requires them to be null together;
the creation/publication operation enforces their correspondence with membership.
`previous_revision_id` is null only for the initial revision, as enforced by the
creation/publication operations. Composite foreign keys prevent a reference from
crossing graphs or selecting another subgraph's publication. The membership
primary key prevents duplicate subgraphs within one revision.

The unique previous-revision constraint permits at most one initial revision and
one successor to each revision. It adds one unique index and encodes the required
linear accepted history directly. It does not replace the function's current-
revision check or prove that membership is complete. `NULLS NOT DISTINCT` requires
PostgreSQL 15 or later; the experiment used PostgreSQL 18.6. The supplied database
version still needs confirmation before implementing a migration.

The graph's current-revision foreign key can be deferred until transaction commit
to support atomic graph creation with its first valid revision. Its column remains
non-null: creation must not leave a committed graph without a revision. Ordinary
publication requires an existing graph; it cannot silently bootstrap missing
history. Graph creation remains a separately authorized operation.

Use `CHECK` for conditions on a row's own values, such as allowed byte lengths
once the actual input limits are selected. Do not hide queries of other rows
inside `CHECK` functions. PostgreSQL documents the limits of cross-row checks
and provides foreign keys and unique constraints for relational integrity.
[PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).

Store exact SDL and generated artifacts as `bytea`. Compute artifact digests from
the stored bytes, using PostgreSQL's binary hash functions if convenient, rather
than trusting a separately supplied digest. JSONB is appropriate for structured
diagnostics and flexible provenance/configuration fields; graph membership and
publication identities remain relational. Original input bytes that must be
retained exactly are not reconstructed from JSONB.
[Binary data](https://www.postgresql.org/docs/current/datatype-binary.html),
[binary hash functions](https://www.postgresql.org/docs/current/functions-binarystring.html),
[JSON storage](https://www.postgresql.org/docs/current/datatype-json.html).

## One database publication function

The application loads an immutable revision, applies authorized requested changes,
and runs the checker before opening the publication transaction. It supplies the
evaluated registry revision, original request, and generated result to a narrowly
scoped PostgreSQL function.

At `READ COMMITTED`, the function:

1. Locks the graph row with `SELECT ... FOR UPDATE`.
2. In a separate statement after acquiring that lock, looks up the idempotency
   key. If already accepted, verifies the original request binding and returns
   that revision. Changed request contents fail without modifying any record.
3. Verifies that the current registry revision is the evaluated revision, and
   that replacements/removals still refer to their expected publications and
   additions are absent.
4. Allocates a fresh registry revision and inserts the complete revision result
   and any new subgraph publications. It derives membership from the previous
   revision plus explicit additions, replacements and removals, preserving all
   untouched publications. Removal omits membership from the new revision;
   it does not delete the historical publication or its earlier memberships.
5. Advances `graphs.current_revision_id` to that new revision.

All writes commit together. Failed preconditions produce no writes; exceptions
roll back the statement. A function result is provisional until the enclosing
transaction commits. The application must not report successful publication
merely because it received a returned revision ID.

The row lock covers only this short database operation. Composition, tool execution
and external calls occur outside it. Readers can continue using immutable
revisions. Different graphs have separate rows to lock. Successive statements at
`READ COMMITTED` and a `VOLATILE` modifying function allow the post-lock lookup to
observe a preceding publisher's committed request.
[Row locks](https://www.postgresql.org/docs/current/explicit-locking.html),
[transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html),
[function volatility](https://www.postgresql.org/docs/current/xfunc-volatility.html).

An intervening unrelated publication requires the application to reload and
recompose. It preserves the original subgraph preconditions. A changed participant
requires caller resolution; the service must not replace that precondition with
the latest value just to make the write succeed. The entire batch remains intact.

`SERIALIZABLE` does not establish that composition performed before the transaction
used the current graph. The expected revision check is still necessary. A
conditional `UPDATE` is another correct implementation, but requires checking its
affected-row count and rolling back all preceding inserts when it updates nothing.
Zero affected rows are not an SQL error. The short row-lock sequence establishes
eligibility before writing and makes that failure path easier to reason about.
[UPDATE behavior](https://www.postgresql.org/docs/current/sql-update.html).

## Empty graphs

Graph creation can commit an initial registry revision with zero membership rows.
Removing the last subgraph similarly creates an empty revision. Both retain the
graph's identity and a non-null current revision; an empty graph is distinct from
a missing graph. The same four relations support these transitions.

For an empty revision, store SQL `NULL` for both composed artifacts and expose
their absence explicitly. Do not substitute zero-byte SDL or reuse an earlier
revision's artifact. Input validation and authorization still apply; composition
is reported as unavailable for an empty graph, not as a successful Federation
composition. Comparisons needing an absent API schema are also unavailable,
not successful compatibility checks. The membership removal remains visible
in the change report.

Direct experiments with `@apollo/composition@2.14.4`, `@apollo/gateway@2.14.4`
and `graphql@16.14.2` found that zero-subgraph composition fails with `NO_QUERIES`.
Gateway's local and introspection composition paths reject empty service lists;
empty SDL also fails. An empty `Query` object is not a valid workaround: GraphQL
requires a query root and at least one field on every object. This supports
absent artifacts rather than a synthetic service or placeholder field.
[Apollo composition errors](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/errors),
[GraphQL root types](https://spec.graphql.org/September2025/#sec-Root-Operation-Types),
[object validation](https://spec.graphql.org/September2025/#sec-Objects.Type-Validation).

An empty registry revision is therefore not selectable as an executable gateway
artifact. Historical artifacts remain retrievable, and an already configured
gateway keeps its explicitly selected revision. Removing registry membership
does not stop that gateway or make its deployed subgraphs unnecessary. Adding
subgraphs resumes ordinary composition and artifact production. The experiments
do not establish behavior for the eventual Rust composition integration or an
unidentified deployed gateway version.

## Idempotency and immutable history

Bind the idempotency key to the graph, authorized submitting identity, requested
additions/replacements/removals, source claims and original subgraph conditions.
Authenticate the caller and authorize access to a recovered result; possession
of a key is not authority. Do not include the discovered registry baseline or newly
composed outputs: those can change legitimately while retrying after an unrelated
publication. Retain and compare the actual normalized request, or derive its
fingerprint inside the trusted operation; do not accept an unchecked caller hash
as proof that two requests match. The request representation is an interface
detail to fix before implementing the function, not another persistent lifecycle.
The function derives inserted subgraph inputs from this same bound request;
it must not accept an independent change list that could disagree with it.

After an uncertain commit, a retry uses the same key. The function's lock and
lookup resolve a competing original transaction and return any accepted result,
even when later publications are current. An unreachable writer leaves the outcome
unknown; an absent result from a lagging replica does not establish failure.
Rejected requests need no durable receipt or reserved idempotency key.

The application role can read and execute the publication function, but has no
direct `INSERT`, `UPDATE`, `DELETE` or `TRUNCATE` privileges on these tables.
Revoking only updates/deletes would still allow it to add a subgraph to an old
revision. A small `SECURITY DEFINER` function earns its cost by exposing the
complete operation while denying these partial mutations.

Use a separate non-login owning role, a trusted `search_path`, schema-qualified
object references, and explicit execution grants after revoking default `PUBLIC`
execution. The runtime role must not own or inherit ownership of the tables.
Owners and database administrators remain trusted; this is not tamper-proof
storage against an administrator. No generic trigger framework or row-level
security policy is needed for the current single trusted application boundary.
[Function security](https://www.postgresql.org/docs/current/sql-createfunction.html#SQL-CREATEFUNCTION-SECURITY),
[database privileges](https://www.postgresql.org/docs/current/ddl-priv.html).

## What PostgreSQL does not establish

PostgreSQL can enforce reference integrity, uniqueness, atomic publication and
the permitted mutation interface. The application still authenticates identities,
authorizes subgraph changes and result access, validates external input, and
establishes Federation validity. It must pass the exact evaluated inputs and
outputs to the database. SQL constraints cannot prove that arbitrary supplied
supergraph bytes were composed from the recorded subgraphs.

The previous source-order and configuration-lifetime proposals remain explicit
policy choices; this data model does not imply Git ancestry enforcement or
instantaneous revocation of admitted work. Gateway activation remains separate
from the current registry revision.

Database durability also depends on WAL/storage settings and the chosen backup
and recovery arrangements. Neither transaction isolation nor idempotency can
recover accepted records lost by restoring an older database.
[WAL settings](https://www.postgresql.org/docs/current/runtime-config-wal.html).

## Verification

Two independent agents assessed relational integrity and transaction behavior.
An isolated local PostgreSQL 18.6 container, without network access or published
ports and using temporary storage, exercised a four-relation fixture and the
restricted publication function. Observed results:

- Two overlapping different requests evaluated against the same revision accepted
  exactly one. Matching concurrent requests returned the same accepted revision.
- Replay after a later publication returned the original revision and output
  without changing the current revision. Changed request contents were rejected.
- A fresh graph baseline did not refresh a stale subgraph precondition. An
  unrelated publication preserved the other subgraph's publication reference.
- Explicit rollback and an error after inserting the first batch participant
  left no partial revision, publication or membership.
- Runtime writes to historical records, insertion of historical membership,
  current-revision rewind and table truncation were denied. Cross-graph current
  revision and publication membership violated their foreign keys.
- Binary output and UTF-8 SDL with CRLF survived exact byte round trips.
- A separate UUID-based creation fixture committed graph and initial revision
  together using the deferred foreign key. Missing revision and null current
  revision cases failed without leaving an incomplete graph.
- The unique previous-revision constraint rejected a second initial revision and
  a second successor, including through direct owner-level inserts.
- An empty-graph fixture exercised creation, addition, removal back to empty,
  and later recreation. Empty revisions used null artifacts; historical inputs,
  memberships and exact artifact bytes remained available. SQL-to-JSON conversion
  represented absent artifacts as JSON `null`.
- Under the agreed create-if-absent rule, additions retried unchanged requests
  across unrelated graph changes. Concurrent same-name additions accepted only
  one request; the other first encountered a changed graph revision, then a
  presence conflict after reloading.
- Recovery of an accepted addition after removal returned its original revision
  without recreating membership. Old replacement/removal conditions failed
  after same-name recreation. Mixed addition/removal batches remained atomic.
- A never-accepted delayed addition succeeded after an intervening creation and
  removal. This verifies the stated limit of ordinary create-if-absent, not
  protection against that sequence. An earlier experiment also exercised the
  stricter fixed-graph alternative, which was not selected.

The main transaction fixture used integer revision labels and a nullable bootstrap
column for convenient setup; the separate creation fixture checked the proposed
UUID/non-null/deferred-reference combination. Neither fixture is a production
migration. Request binding used a concrete JSONB fixture request; the final
application request representation is not yet specified or verified. Composition
outputs were supplied fixture bytes, and recomposition was represented by
resubmitting against a new evaluated revision. Actual response loss, HTTP,
authentication, source-release selection, crashes, failover and backup/restore
were not tested. These experiments support database-operation claims only;
the separate library startup/composition results and their limits are recorded
under [empty graphs](#empty-graphs).
