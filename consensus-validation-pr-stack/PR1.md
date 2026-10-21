# PR1 — authentication, binding and preparation

**Repository PR #17.** Implementation exists; this page does not assert current merge or test readiness.

## Responsibility

Produce a protected `PreparedExecutionProof` from the signed message and one trusted Store snapshot. No Seen insertion, engine call, verified storage or final ACCEPT.

Preparation order:

1. Reject empty data or unsupported protocol type.
2. IGNORE unknown B or an already verified `(B,T)`.
3. Compute bare-envelope root R; IGNORE Seen root/prover keys.
4. Obtain the accepted full payload and B’s exact trusted post-state.
5. Authenticate the envelope, reconstruct engine input and create Prepared.

Protocol types are runtime checked; bounded ByteList enforces the upper size limit during construction/decode.

## Design conclusions and alternatives

- **Exact trusted context:** require both acceptance marker and matching full envelope. Fallibly reload/reconstruct B’s state rather than treating eviction as invalidity. Head/same-slot/next-slot substitution is incorrect; resident-only loading loses availability, while broad legacy-loader migration expands scope.
- **Validate before cache publication:** trusted replay can install a declared cached root. Independently check state contents before publishing a new cache fill, and check resident/cache-hit states too. Post-insertion validation could reject a state after other readers received it; earlier validation costs additional cache-lock work.
- **Protected conventional module:** private construction and read-only access in one ordinary preparation module. Reject top-of-Store, nested `store/` and path-attribute workarounds. Consistency checks do not replace authentication.
- **Semantic identity:** B plus full `PublicInput`, not pointer identity. PR2 must still prove live applicability.
- **Named borrowed precheck token:** carry R while borrowing the exact input, preventing token-A/envelope-B substitution without rehashing. Inline continuation fields could be simpler for one consumer; later helpers benefit from a named bundle. Prepared itself owns its data.
- **Typed outcomes:** protocol validation failures REJECT; spec duplicates/unavailability IGNORE. Materialization failure maps to local IGNORE; binding/consistency errors are local errors, not automatic peer rejection.
- **Deterministic Store signing:** test-only `bls` generates coherent signatures as fixture inputs change. This replaced print-and-freeze Store fixtures; fixed helper signing vectors remain to catch shared sign/verify bugs.

## Evidence required

Ordering/precedence, authentication failures, binding field selection, exact-state loading/cache safety, protected provenance and size/decode bounds. Tests must show no proof-bookkeeping mutation or engine execution. Keep preparation regressions, not investigation-only payload-cache experiments. Shared state/pubkey cache warming is allowed; preparation is not globally side-effect-free.
