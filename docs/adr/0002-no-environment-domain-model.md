# Keep deployment environments outside the domain model

Selvedge does not assign built-in meaning to deployment environments or model
promotion between them. Each independently composed graph has its own
configuration; directory names and repository organization are operator choices,
avoiding an environment hierarchy that would constrain those choices. The initial
storage convention is recorded separately in
[ADR 0005](0005-graph-directory-input.md).
