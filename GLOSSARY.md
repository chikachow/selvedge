# Federation Schema Registry

Language for reviewing schema changes across a federated GraphQL API. These
definitions carry forward the project handoff; open design choices are recorded
in [the design discussion](docs/design.md).

## Language

**Subgraph**:
A GraphQL service's contribution to a federated graph, expressed by its
Federation schema.

**Supergraph**:
The composed representation of a federated graph, including information needed
to route operations across its subgraphs.

**Public API schema**:
The client-facing GraphQL schema exposed by the composed graph.
_Avoid_: Supergraph SDL, when referring only to the client-facing type system.

**Graph configuration**:
The complete collection of subgraph schemas and configuration composed together
to produce a supergraph.

**Source PR**:
A proposed code change in a subgraph's source repository, from which candidate
Federation SDL is generated.

**Registry PR**:
A proposed graph-configuration change in the registry repository.

**Subgraph source revision**:
The code revision used to generate a subgraph's candidate SDL.

**Registry revision**:
An immutable record of an accepted graph configuration and the artifacts
generated from it.

**Proposed configuration**:
A candidate collection of subgraph schemas and configuration submitted for
review.

**Approved configuration**:
A graph configuration accepted into the registry under the configured source
and registry repository controls.
_Avoid_: Deployed configuration, active configuration.

**Observed schema**:
The schema available from a deployed subgraph at the time it is observed.
_Avoid_: Approved schema, when describing runtime evidence.

**Supergraph artifact**:
A generated supergraph retained for selection by a gateway deployment.

**Subgraph publication**:
Acceptance of a subgraph schema into the registry's graph configuration.

**Publication intent**:
The exact requested registry update and its provenance, kept stable across
retries even when the evaluated registry revision changes.

**Publication receipt**:
A record identifying an accepted publication and the exact inputs and result
to which that acceptance applies.

**Supergraph artifact publication**:
Making a generated supergraph artifact available for later deployment selection.
_Avoid_: Activation, approval.

**Validity result**:
The assessment of whether analysis inputs can be processed and the complete
federation composes under the selected tools.

**Compatibility result**:
The assessment of public API changes and known-operation validity changes
between identified baseline and candidate revisions.

**Activation**:
The adoption of a supergraph revision by a gateway for execution.
_Avoid_: Approval, publication.

**Active supergraph revision**:
The supergraph revision a gateway is currently configured to use for execution.
_Avoid_: Latest approved revision, when describing gateway state.

**Known client operations**:
The collection of client GraphQL operation documents supplied for validation.
It does not necessarily include every operation clients may execute.
