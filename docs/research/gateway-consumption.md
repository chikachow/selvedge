# Gateway consumption of approved supergraphs

Research date: 2026-11-01. This supports the [design discussion](../design.md)
and uses the [project glossary](../../GLOSSARY.md). Recommendations below are
proposals, not accepted release policy. The existing gateway is described as
Apollo's JavaScript Gateway using `IntrospectAndCompose`; its installed version
and configuration have not been inspected. This is documentation and source
research, not an executed integration test.

## What changes when introspection is replaced

The desired boundary is achievable: the gateway can execute a previously
composed supergraph supplied from the registry's artifacts, including artifacts
stored in its backing Git repository. A registry application server is not
required merely to provide the SDL.

A backing repository containing only subgraph SDL is not already a distributable
supergraph. Composition must produce and persist the generated supergraph bytes;
reading source files from Git does not perform that step or serve an artifact.

Apollo documents startup composition failures and inconsistent schemas between
gateway instances as limitations of `IntrospectAndCompose`. Its documented
alternative supplies composed SDL through `supergraphSdl`, either statically or
with a custom update hook. The static form requires restarting the gateway to
change the schema. The hook initializes once and returns the initial SDL; the
application can subsequently call its supplied `update` function.
[Apollo Gateway setup](https://www.apollographql.com/docs/apollo-server/using-federation/apollo-gateway-setup#updating-the-supergraph-schema).

The following consequences are design inferences from that execution boundary:

| Outcome | What supplying an approved artifact establishes | Remaining limit |
| --- | --- | --- |
| Approval controls schema exposure | A new field found in a live subgraph cannot automatically enter the gateway's public API through introspection. | This depends on selecting only approved artifacts and removing the introspection fallback. It does not prevent direct subgraph access. |
| Stable composition input | The gateway need not assemble a potentially mixed collection of live subgraph schemas. | Different gateway instances can still select different artifact revisions during rollout. |
| Review matches execution configuration | The selected artifact can be the same bytes produced and checked before activation. | Recomposition with different tools, substituted routes, or a moving branch read can break that correspondence. |
| Changes to implementations remain possible | The gateway keeps its selected schema until activation changes it. | A subgraph can remove a needed field, change authorization, or alter resolver behavior underneath that unchanged schema. Approval of SDL does not prevent those changes. |

Thus this changes the enforcement question: registry approval can govern the
gateway's execution configuration without governing every subgraph deployment.
It does not by itself establish that the approved configuration is executable.

## Apollo's integration surface and its limits

`supergraphSdl` accepts an SDL string, an asynchronous hook, or a
`SupergraphManager` with an `initialize` method. The hook receives `update`,
`healthCheck`, and `getDataSource`; it returns initial SDL and optionally an
asynchronous `cleanup` function. The SDL includes subgraph routing information.
The general `fetcher` option configures Apollo managed-federation fetching; it
does not turn an arbitrary Git URL into an SDL source.
[Gateway API reference](https://www.apollographql.com/docs/apollo-server/using-federation/api/apollo-gateway#supergraphsdl),
[hook types](https://github.com/apollographql/federation/blob/main/gateway-js/src/config.ts).

Consequently, “consume directly from Git” still needs a small adapter: retrieve
the chosen revision's artifact bytes, authenticate the source as appropriate,
then provide the SDL string. This can happen before startup, including when
building a gateway deployment, or inside a custom hook. Passing a Git URL as the
`supergraphSdl` string would pass a URL where schema text is expected.

Apollo's update example explicitly calls `healthCheck(candidate)` before
`update(candidate)`, catches errors, and describes continuing with the existing
schema. Its cleanup callback stops the watcher when the gateway stops. This is
an example of application-owned update handling, not a complete distribution
or durable recovery protocol.
[Apollo dynamic-update example](https://www.apollographql.com/docs/apollo-server/using-federation/apollo-gateway-setup#updating-the-supergraph-schema).

The gateway source sends `query __ApolloServiceHealthCheck__ { __typename }` to
the candidate's subgraph endpoints. It initializes the manager and then applies
its returned SDL; initialization failure rejects loading. Parsing/building a
candidate can throw before the main schema assignment, but later update stages
and application callbacks also exist. A blanket claim that every possible
update error atomically restores all previous state is not established here.
[Gateway initialization, update, and health-check source](https://github.com/apollographql/federation/blob/main/gateway-js/src/index.ts).

**Inference:** this health check demonstrates limited endpoint responsiveness.
It does not prove that all replicas serve the candidate schema, that required
entity representations resolve correctly, or that authorization and resolver
behavior remain compatible. A single successful request through a load
balancer cannot establish all-replica readiness.

The current `IntrospectAndCompose` source catches polling failures and schedules
another poll, but rethrows initialization failures. Removing introspection
does not automatically supply replacement retry or recovery behavior.
[IntrospectAndCompose source](https://github.com/apollographql/federation/blob/main/gateway-js/src/supergraphManagers/IntrospectAndCompose/index.ts).

**Proposed guarantee if dynamic updates are required:** reject unsuccessful
fetches, unauthorized artifacts, and failed readiness checks before calling
`update`; report the revision actually installed. Persist any fallback artifact
needed across process restarts. An in-memory last successful revision cannot
bootstrap a new instance during a distribution outage. Define and test update
failure handling against the gateway version actually deployed; do not infer
durable recovery from Apollo's example.

## Approval and activation are different choices

Git distinguishes object identities from symbolic reference names such as
branches. An adapter should resolve a moving reference once and fetch the
artifact and metadata from that resolved revision, rather than combining files
read at different branch heads.
[Git revision syntax](https://git-scm.com/docs/gitrevisions#_specifying_revisions).

These are alternative policies, not three required implementation stages:

| Policy | Activation event | Benefit | Consequence to accept |
| --- | --- | --- | --- |
| Manual immutable revision | An operator selects a specific approved artifact in gateway configuration or a deployment. | Approval can precede readiness; selection and rollback targets are explicit. | Updates need an operational step; running instances can lag approval. |
| Automatically follow the approved branch | A loader notices an approved branch revision and selects its artifact. | Minimal separate activation workflow. | Approval becomes permission for eventual runtime exposure. Merging before required deployments are ready can break execution; polling is not a fleet-wide atomic switch. |
| Separate activation reference | A separately updated reference selects one already approved artifact. | Approval and readiness can advance independently; moving the reference can select a previous retained artifact. | Adds activation authority, reference-update rules, and observation of gateway adoption. The reference itself is not proof that every gateway has switched. |

The consequences in this table are design analysis, not features supplied by
Git or Apollo. A service, Git reference, or static manifest could express the
selection; the required authority and readiness semantics should decide the
mechanism.

## Deployment ordering and rollback constraints

Apollo recommends deploying backward-compatible subgraph changes, waiting for
all replicas, and then publishing the schema so the gateway cannot request
fields absent from older replicas. It also documents exceptions: a `@requires`
migration can need a resolver accepting both old and new representations before
changing the registered schema; removing an interface implementation can require
changing the registered schema before removing support in the subgraph.
[Apollo deployment guidance](https://www.apollographql.com/docs/graphos/platform/production-readiness/deployment-best-practices#updating-subgraphs-safely).

**Inference for this project:** the corresponding exposure event is activation
of the composed artifact, not merely approval in Git. The required order depends
on the migration. Validating the final combined configuration does not establish
that every intermediate deployment state works.

**Rollback constraint:** retaining an old artifact preserves the ability to
select it, not the ability of current subgraphs to execute it. Once a field,
entity representation, endpoint, or compatible resolver path is removed, the old
artifact may no longer be usable. A promised rollback window therefore needs
matching retained runtime support, or a coordinated restoration procedure.
This follows from the intermediate compatibility requirements in
[Apollo's migration examples](https://www.apollographql.com/docs/graphos/platform/production-readiness/deployment-best-practices#changes-affecting-query-planner-performance).

## Smallest initial distribution proposal

**Recommendation, pending activation requirements:** begin without a registry
server or custom live-update loop. Persist each accepted composed SDL and its
provenance; have a gateway deployment read or fetch an explicitly selected
artifact and supply it through static `supergraphSdl`. Git storage is sufficient
if it meets artifact access and retention needs. Existing static artifact
storage is another option; neither implies a new database or application server.

Keep the artifact bytes associated with the complete input revision, graph
configuration identity, composition tool version, resolved routing configuration, and
validation result. A digest identifies the bytes; it does not independently
prove approval or trusted production. The approval record must identify the
artifact or the trusted production of that artifact from the accepted inputs.
Reading a raw file from a protected branch alone should not be described as
proof that those bytes passed the intended checks.

Pin the composition version and check compatibility with the deployed gateway.
Apollo explicitly warns that a floating composition version can change output
without input changes and that the composition version must not exceed router
support.
[Rover composition versions](https://www.apollographql.com/docs/rover/commands/supergraphs#setting-a-composition-version).

For restart recovery, bundling the selected artifact into the gateway
deployment avoids needing a remote artifact fetch on every startup. If startup
fetching is preferred, specify what happens when the source is unavailable and
whether a retained artifact may be used. Keep artifacts and deployment
references for the chosen rollback period; the period and runtime compatibility
obligation remain undecided.

Add a custom updater only if schema changes without gateway restart are a
requirement. Add a distinct activation reference only if approval must precede
runtime selection and manual deployment selection is insufficient. Neither
requires a custom server by itself.

The interview still needs to settle whether approval should activate a schema,
what evidence makes an approved revision ready, and whether runtime selection
must change without restarting the gateway. The actual gateway dependency and
failure/recovery behavior then need an integration check; no implementation or
runtime verification has been performed for this research.
