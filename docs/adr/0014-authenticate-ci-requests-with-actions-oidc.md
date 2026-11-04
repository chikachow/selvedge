# Authenticate CI requests with GitHub Actions OIDC

The first release authenticates source and maintainer CI requests using
short-lived GitHub Actions OIDC credentials. Explicit repository, workflow,
event/ref, and operation/subgraph grants distinguish checking from publication
authority, avoiding a long-lived Selvedge credential in each source repository.
The service's GitHub App reports source checks and performs authorized registry
writes; its write credentials remain inside the service.

OIDC establishes authenticated job context. Selvedge still validates request
scope and binds the received bytes to the generated result; job authentication
does not prove source-to-SDL linkage or runtime state. Concrete trust settings,
token and check association behavior, and eventual repository permissions need
integration verification. Support for other CI authentication mechanisms is not
part of this initial decision.
