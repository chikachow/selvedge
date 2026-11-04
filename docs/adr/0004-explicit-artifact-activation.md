# Select the supergraph through gateway deployment configuration

The gateway initially consumes an explicitly selected approved supergraph
artifact through its deployment configuration. This allows approval to precede
deployment readiness and avoids making a merge an automatic activation
instruction, at the cost of a separate deployment selection step. Artifact
storage now follows [ADR 0010](0010-retain-validated-outputs-with-acceptance.md);
the retrieval interface remains to be specified. This activation boundary does
not require live polling.
