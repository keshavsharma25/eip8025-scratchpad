# PR1 decisions — Grandine PR #17

PR1 prepares an authenticated, payload-bound execution proof for later admission. This summary covers the selected design and implementation rationale, not current verification or merge readiness. **Recommended approaches are highlighted**; implementation details are not all separate operator choices.

## 1. Preparation belongs in PR1

**Conclusion:** expand PR1 beyond authentication/binding helpers to include trusted-context selection and a protected prepared result. PR2 owns live admission, completion and pruning; PR3 owns task/engine integration.

**Recommended rationale:** preparation is a coherent, independently testable boundary. It authenticates and reconstructs the expected statement without consuming Seen keys, invoking the engine, storing a verified proof or returning final ACCEPT.

**Alternative:** leave preparation in a separate PR2a. This keeps PR1 smaller but adds another stack boundary between the helpers and their usable result.

## 2. Detect context changes instead of changing payload acceptance

**Selected: Option B — semantic context checks.** Preserve the prepared statement identity and require live applicability checks before admission and at completion.

**Recommended rationale:** keep proof validation separate from existing Gloas payload-acceptance policy. Equivalent content in different allocations should remain equivalent; changed context should cause a local decline/discard.

**Alternative: Option A — prevent replacement in the payload path.** Recheck payload eligibility/duplicates before applying results. This addresses replacement at its source but expands scope into existing payload-processing behavior.

**Implementation:** identity is the target beacon block root plus the full reconstructed `PublicInput`, rather than pointer identity or a general generation framework. Identity alone does not establish live applicability: PR2 still needs trustworthy, synchronized current-context evidence without expensive mutator work.

## 3. Keep the selected binding baseline

**Conclusion:** retain PR18’s progressive `VersionedHashes` representation and the selected progressive-list limit semantics. Use the master baseline for authentication/gossip ordering; defer #5729 rather than implement speculative compatibility.

**Recommended rationale:** preserve the agreed data-model baseline and isolate binding so later alignment remains narrow.

**Alternative:** revert to ordinary master lists or adopt anticipated spec changes immediately. The former undoes the selected representation; the latter builds against unsettled behavior. The retained binding target must not be described as pure-master interoperability.

## 4. Load exact trusted context, not a convenient substitute

**Implementation conclusion:** require the target block’s exact validated post-state, its accepted-payload marker and the matching full payload envelope. Use fallible loading/reconstruction when the state is not resident.

**Rationale:** a marker does not supply payload bytes, cache presence does not establish acceptance, and state eviction does not make a block invalid. Head state, another same-slot block and next-slot state are not substitutes. Unavailable/inconsistent materialized context gives local IGNORE.

**Alternatives considered during engineering analysis:** resident-only IGNORE is simpler but loses availability; reusing infallible loading risks panics; migrating every legacy loader or duplicating replay broadens scope. The chosen decomposition keeps loading mechanics in Store.

## 5. Validate state before cache publication

**Corrected implementation:** independently validate reconstructed target contents inside the cache-fill closure before publishing them; also validate resident states and cache hits.

**Rationale:** trusted replay can install a declared outer `Hc` root, so reading that cached root is not independent content validation.

**Earlier approach:** validate only after cache insertion. Preparation could reject an inconsistent state after other readers could already access it. Moving validation earlier closes that publication gap, at the cost of more validation work under the cache lock. It does not guarantee eviction of existing bad entries.

## 6. Protect provenance using conventional module boundaries

**Accepted layout:** one `prepared_execution_proof.rs` implementation file, declared normally as a private module with selective re-exports.

**Rationale:** private construction and read-only access preserve the relationship between the authenticated envelope and reconstructed input. Avoid accessors that expose interior cached-root mutation. Consistency checks support this boundary but do not replace authentication.

**Rejected arrangements:** top-of-`store.rs` carrier code, a nested `store/` layout and a `#[path]` workaround. They obscured the boundary or conflicted with local conventions.

## 7. Carry precheck results without repeating or mixing work

**Implementation:** a named `PrecheckedExecutionProof<'a>` borrows the exact signed input and carries its computed bare-envelope root. The final Prepared value owns its data.

**Rationale:** later helpers can consume one bundle, and cached root A cannot accidentally be paired with envelope B. Rehashing would detect that mismatch but repeat potentially expensive work.

**Alternatives discussed:** inline `Continue { … }` fields could be simpler for a single immediate consumer; a separate metadata token plus another envelope requires an additional consistency check. `Result<()>` alone cannot distinguish IGNORE from continuation. These interface details were implementation choices, not separately selected protocol requirements.

## 8. Separate protocol invalidity from local failure

**Conclusion:** preserve typed REJECT, IGNORE and continuation outcomes; continuation is not ACCEPT.

**Rationale:** invalid message structure/authentication warrants protocol REJECT. Unknown blocks, verified proofs, Seen duplicates and unavailable payloads follow spec IGNORE rules. Grandine maps unavailable/inconsistent materialized context to local IGNORE; binding/construction errors remain local errors, not automatic peer rejection.

**Avoid:** treating every Rust `Err` as peer-invalidity or collapsing IGNORE and continuation into one success result. State loading, BLS and payload hashing remain off the mutator.

## 9. Use dynamic Store signatures and retain fixed vectors

**Selected:** add test-only `bls` to `fork_choice_store` and generate deterministic signatures for coherent Store fixtures.

**Final recommended rationale:** one dynamic approach makes fixtures easier to maintain as block roots and inputs change. Keep fixed signing-root/public-key/signature assertions in `helper_functions` so sign-and-verify cannot hide a shared bug.

**Earlier recommendation:** avoid the dependency and generate/freeze Store fixtures. This reduced dependencies but introduced a manual regeneration workflow; it was superseded by the selected dynamic approach.

## 10. Keep implementation scope and evidence separate

**Conclusions:** use seven reviewable checkpoints with discussion stops and tests alongside implementation; remove investigation-only payload-cache tests while retaining preparation regressions. Keep independent binding vectors and real-backend interoperability in a separate workstream, outside PR2/PR3 implementation scope.

**Rationale:** distinguish exploratory evidence from durable contract tests, and avoid expanding lifecycle/integration work into interoperability research. Independent vectors still gate interoperability sign-off; Grandine-generated expected roots do not replace them.
