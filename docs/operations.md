# Registry operations design

Status: design requirements, 2026-11-04. These workflows guide implementation;
there are no runnable deployment or migration commands in this repository yet.
The [system design](design.md) owns application behavior, and the
[data core](postgresql-data-core.md) owns database invariants and privileges.

## Gateway deployment

The default is to distribute a selected artifact with the gateway deployment.
Retrieval happens during deployment preparation, before the gateway process
starts. This removes registry availability and registry credentials from cold
startup and request execution; it moves availability and access-control duties
to the deployment preparation and distribution systems.

1. Select a nonempty accepted registry revision explicitly. Obtain its retained
   artifact and metadata over authenticated HTTPS using a read-only deployment
   preparation identity. Do not recompose or resolve a mutable `latest` pointer
   at startup.
2. Verify graph identity, revision identity, artifact kind and the digest of the
   exact bytes. Package the bytes and this metadata together in the deployment's
   immutable distribution unit. The digest detects mismatched/corrupt bytes;
   authenticated retrieval and trusted distribution establish provenance.
3. At startup, verify the local bundle and load the selected artifact with the
   pinned compatible gateway. Become ready only after initialization and the
   deployment's required subgraph/readiness checks succeed.
4. Activate by rollout/restart. Retain the prior running deployment until the new
   one is ready. Record the artifact identity actually loaded, so operators can
   distinguish registry publication from gateway activation.

For Apollo JavaScript Gateway, retained SDL can be supplied via `supergraphSdl`.
That interface is documented; compatibility of the eventual composition output,
gateway version and deployment configuration still needs an integration test.
[Apollo Gateway interface](https://www.apollographql.com/docs/apollo-server/using-federation/api/apollo-gateway).

| Failure or transition | Required behavior |
| --- | --- |
| Registry unavailable during preparation | Do not produce a new bundle unless an already verified copy of the exact selected artifact is available. |
| Registry unavailable at cold start or while serving | A complete verified bundle remains loadable without registry access; existing gateways continue with their loaded graph. |
| Missing, corrupt, wrong-revision or unsupported artifact | Fail startup/readiness. Do not substitute another revision; preserve the existing deployment during rollout. |
| Empty registry revision selected | Reject it as a gateway artifact; it has no executable supergraph. |
| Historical artifact selected for rollback | Check that deployed subgraphs still support it. Retained bytes do not prove operational rollback is possible. |
| Artifact read grant revoked | Block future registry reads under that grant. Already distributed or loaded bytes cannot be retracted by registry read authorization. |

The gateway has no registry polling, live update manager or last-successful cache
fallback in this initial model. Deployment owns bundle storage/access controls,
distribution reliability and retention. Removing a subgraph from the registry
does not stop a gateway still using an older artifact. Subgraph sequencing and
runtime health remain deployment responsibilities.

## Writer and control handover

The initial release model admits one pinned checker and authorization-control
release for publication at a time. A release can have multiple service instances;
they must share the same admitted controls. Checker identity in a receipt records
evidence but does not itself prevent an obsolete writer from accepting work.

Use an exclusive handover:

1. Evaluate the proposed checker/options/control change separately against retained
   graph inputs, stored outputs and the applicable operation corpus. Bind evidence
   to the exact graph/corpus revisions, proposed release and options; preserve
   changed artifacts and diagnostics. Existing controls authorize the exact
   proposed change; source requests cannot install
   their own rules. Complete the repository approval association required by the
   [open approval contract](design.md#unanswered-policy-questions).
2. Stop admission of new publication requests on every old writer. Keep authorized
   receipt/revision reads available where feasible. A stopped HTTP listener alone
   does not establish that database writes have stopped.
3. Drain bounded admitted work, reconcile uncertain commits and account for all
   old write-capable processes and database sessions. Work admitted before closure
   may finish under the old release during draining. A timeout, terminated caller
   or cancelled request is not proof of transaction rollback.
4. Verify that old writers can no longer execute publication and no old transaction
   can commit later. If that cannot be established, keep publication closed. The
   deployment must provide a tested stop/credential/session-control mechanism;
   the design does not assume an HTTP cancel can fence a writer.
5. With old writes excluded, identify the final current revision of every affected
   graph. If any graph/corpus or proposed release input differs from the evaluated
   set, repeat the upgrade assessment against those final retained inputs and
   outputs. Do not reopen with evidence for an obsolete head. Complete any required
   approval of the final assessment under the existing admission controls.
6. Activate the exact assessed release, verify its database/schema compatibility
   and authority configuration. If read grants change, close old read admission,
   drain or end affected in-flight reads, and switch all serving readers to the
   new grants before declaring read revocation effective. Stop readers that cannot
   transition, including instances or caches serving receipt/revision reads.
   Reopen publication only after these checks. An unsuccessful assessment
   leaves publication closed until an authorized release is safe to resume.
   Retried requests recover prior receipts before new evaluation under the new
   release.

This defines the revocation boundary: a control change is fully effective for new
acceptance only once old writers and their pending transactions are excluded.
Read revocation additionally requires the reader transition above; writer exclusion
alone cannot prevent disclosure by a stale reader.
Emergency suspension closes admission and prevents further writes using the same
verified process, with reconciliation for any uncertain completion. It cannot
undo an already committed publication or retract a distributed artifact.

The trade-off is publication downtime and an operational exclusion requirement,
in place of distributed epochs, mixed-version admission and cancellation fencing.
The operator must confirm those assumptions before implementation. If zero-downtime
publication or uncontrolled old writers are required, this handover is insufficient;
design and verify a different fencing protocol before enabling that topology.
Ordinary read availability during a schema change is not guaranteed.

Checker rollout does not rewrite historical artifacts, receipts or compatibility
evidence. Subsequent publication compares against the retained baseline under the
[analysis contract](design.md#static-validation-and-comparison). Rolling back the
service does not roll back accepted history: the old release must understand all
retained formats and the live database before it can resume writes. Otherwise
remain suspended and repair forward.

## Database schema management

Use [pgschema](https://www.pgschema.com/) with version-controlled desired SQL.
Desired SQL is authoritative; target-specific saved plans are deployment artifacts,
not a second manually maintained migration history. Run one explicit operator
workflow. Application startup must not acquire privileged migration credentials
or independently apply schema changes.

| Managed by pgschema | Established outside pgschema |
| --- | --- |
| Supported tables, constraints, functions and explicit object grants | Database/schema creation, roles and membership, object ownership and global default privileges |

pgschema does not manage ownership or global default privileges; such statements
are skipped with warnings. Objects belong to the effective creating role. Its
fingerprint therefore cannot establish the complete administrative security state.
[Supported scope and exclusions](https://www.pgschema.com/syntax/unsupported).

### Operator workflow

1. **Bootstrap and preflight.** Establish the database/schema, non-login owner,
   separate migration identity and restricted runtime role. Validate the mechanism
   that makes the designated owner the effective object-creating role. Confirm
   version pins, recovery readiness, application compatibility and that only one
   schema operation is running. Use the writer handover when the old application
   cannot safely coexist with the change.
2. **Plan.** Generate a saved plan for this target from the reviewed desired SQL
   using the pinned tool. Inspect destructive/replacement operations, DDL locks,
   transaction groups, creation order, backfills and effects on accepted data.
   Record the SQL commit, tool version and target with the plan. A plan for a
   different database or stale target state is not interchangeable.
3. **Apply.** Apply the inspected saved plan with the same pinned tool and
   fingerprint validation enabled. A stale fingerprint requires inspection and
   replanning, not bypassing the check. Schema mutation uses the migration identity,
   never the runtime service identity.
4. **Verify and reopen.** Check convergence against desired SQL and independently
   inspect ownership, function security, grants and effective runtime permissions.
   Exercise publication/recovery and retained-byte reads against the resulting
   schema before reopening affected writers. Preserve plan/apply evidence and
   redact credentials from logs.

The plan/apply split and fingerprint validation are supported by pgschema.
[Planning](https://www.pgschema.com/cli/plan),
[saved-plan application](https://www.pgschema.com/cli/apply).

Before functions are created, trusted bootstrap must revoke global default
`EXECUTE` for `PUBLIC` for the effective object-creating role. Defaults of another
role are not inherited at creation merely through membership; a per-schema revoke
cannot remove a global grant. Explicit function ACLs still need enforcement on
existing objects. The connection/role-switching mechanism remains to be proved
with the selected pgschema version.
[PostgreSQL default privileges](https://www.postgresql.org/docs/18/sql-alterdefaultprivileges.html).

The post-apply privilege check must establish the
[data-core access model](postgresql-data-core.md#idempotency-and-access-control):
trusted ownership, restricted schema creation, qualified function references,
safe `search_path`, `SECURITY DEFINER`, exact execution grants and denied direct
runtime table writes. Check effective inherited privileges as well as explicit
ACL entries. Schema convergence alone is insufficient.

### Partial application and data preservation

A pgschema apply is not globally atomic: transaction groups can commit before a
later failure, and operations such as concurrent indexes run outside transactions.
On failure, inspect actual target state and committed work before replanning.
Keep incompatible writers stopped until the database is known to support them.
The atomicity of one registry publication does not extend to schema deployment.
[Transaction handling](https://www.pgschema.com/cli/apply).

pgschema does not infer renames or manage data transformations. Inspect apparent
drop/recreate changes and design explicit backfills when needed. Reverting desired
SQL cannot restore deleted accepted records. Every migration must preserve receipt
bindings, membership identity, historical readability and exact retained bytes;
destructive changes require a separately authorized lifecycle design rather than
being treated as ordinary convergence.
[Data migrations](https://github.com/pgplex/pgschema/blob/v1.13.4/docs/workflow/data-migration.mdx),
[rollback](https://www.pgschema.com/workflow/rollback).

The current review inspected pgschema v1.13.4 source; it did not execute this model
through pgschema. Before adoption, pin tool/database versions and prove fresh
installation, upgrades, administrative bootstrap, convergence and partial-failure
recovery. Include cyclic foreign-key DDL ordering: deferred row checks do not
defer object-name resolution during DDL. The
[implementation matrix](implementation.md#required-behavioral-evidence) specifies
the acceptance evidence.

## Durability, failover and restoration

Accepted records and exact inputs/outputs have no automatic expiry. This logical
retention policy is distinct from physical durability. Operators must choose and
test write durability, storage, replication, backups, recovery point objective
(RPO) and recovery time objective (RTO). PostgreSQL WAL and commit settings affect
which acknowledged writes survive failure.
[WAL configuration](https://www.postgresql.org/docs/18/runtime-config-wal.html).

Only an authoritative writer may establish receipt presence or observed absence
for reconciliation. Failover must exclude the old writer before the replacement
accepts work; a lagging replica cannot prove a request was never accepted.

After restoration or failover with possible accepted-history loss, suspend
publication and identify the recovery boundary. Reconcile externally observed
receipts, deployment artifact identities and the restored accepted history before
permitting uncertain requests to run again. An absent key in the restored database
may represent a lost acknowledged acceptance. It must not be treated as a new
request automatically. Do not invent missing history from an artifact or silently
reuse a graph identity with reset history.

The reconciliation procedure, required external evidence and acceptable recovery
gap remain deployment prerequisites. If missing accepted records cannot be
recovered or safely reconciled, report the limitation and keep affected writes
suspended until an explicit recovery decision. Ordinary receipt replay cannot
repair lost history. Graph retirement, physical deletion, retention-policy changes
and identifier reuse need their own lifecycle contracts.
