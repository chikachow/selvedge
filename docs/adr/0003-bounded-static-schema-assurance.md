# Bounded static schema assurance

The initial release requires complete Federation composition and public API
change classification, supports validation of an optional supplied operation
corpus, and reports Federation changes for human review. This uses established
analysis engines while keeping operation collection and automated execution-safety
judgments outside the release scope. Missing operation evidence is reported
explicitly; static check results do not establish runtime equivalence.
