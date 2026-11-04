# Snapshot checks and conditional registry updates

Source checks provide evidence for their identified candidate and registry
snapshot. Publication applies the permitted subgraph updates to the latest
registry, composes the complete candidate, and conditionally commits against the
state it validated; an intervening independent update triggers internal refresh
and recomposition rather than a caller-visible stale-revision conflict. This
allows valid independent publications to overlap without introducing
cross-repository merge coordination.
