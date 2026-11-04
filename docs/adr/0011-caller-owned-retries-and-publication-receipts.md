# Caller-owned retries and durable publication receipts

Publication requests have stable identities bound to their exact payloads.
The receipt retained with an accepted snapshot lets a retry return that original
acceptance even after subsequent publications; uncertain writes are reconciled
before replay, and receipt recovery precedes treating an already accepted
request as an obsolete source update. Reusing an identity for another payload,
or producing different bytes from the same declared generation inputs, is an
explicit integrity failure rather than permission to replace accepted output.

A genuinely uncomposable publication leaves accepted state unchanged and returns
diagnostics for the evaluated revision. If the registry advanced during
evaluation, the service reevaluates before returning a semantic rejection.
Rejected requests are not retained for automatic later publication: source CI
owns correction and explicit retry. Internal contention retries are bounded;
exhaustion or unavailable infrastructure is an operational result, and an
unreconciled write remains unverified rather than falsely reported absent.
