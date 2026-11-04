# Make a complete graph directory the core input

The checker operates on one complete graph configuration directory per invocation;
initially each registry repository stores one such configuration at its root.
This aligns the first integration with repository-level checks and exception
authority while avoiding multi-graph dispatch. Repository placement remains an
integration choice, so shared repositories can later contain independent graph
directories without changing GraphQL validation semantics.
