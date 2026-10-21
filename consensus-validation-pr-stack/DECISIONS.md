# Stack decisions

Selected policies for PR1 preparation, PR2 Store lifecycle and PR3 integration. **Selected approaches are highlighted.** Some API details remain implementation choices; the unresolved live-context predicate is not presented as solved.

## 1. Separate preparation, lifecycle and integration

**Selected:** PR1 prepares; PR2 admits/completes/prunes; PR3 orchestrates engine work and terminal results.

**Why:** each boundary can be reviewed and tested independently without wiring a partially enforced service. Preparation means authenticated and bound, not admitted or verified.

**Alternative:** a separate preparation PR2a, or one read-only function that prepares and verifies. The former adds a stack boundary; the latter cannot enforce serialized live admission before engine entry.

## 2. Consume both Seen keys before engine scheduling

**Selected: permanent Seen until target retirement.** One live Store operation checks all conditions and inserts both keys or neither:

- `(B,R)`: target beacon block root and bare-envelope root;
- `(B,T,V)`: target root, proof type and validator index.

**Why:** workers can share an old snapshot. Completion-only updates cannot prevent duplicate engine work. Authentication must precede insertion so forged messages cannot suppress genuine attempts.

**Alternative:** paired pending reservations with ownership tokens and stopped-worker release. This permits local-failure retry but adds recovery and race machinery. It was superseded by the simpler permanent-Seen policy. Invalid verdicts, scheduling failures and local faults do not roll back admission; same-key retry is deliberately unavailable while retained.

## 3. Preserve the boolean engine boundary

**Selected:** `ProofEngine::verify_execution_proof(ExecutionProof) -> bool`; bypass a null engine before preparation/admission.

**Why:** Store lifecycle does not need a new backend API. Applicable false means invalid proof; opt-out and separately reported local failures must not be fabricated as false or peer-invalidity.

**Alternative:** add operational-error, cancellation and recovery APIs now. Those are separate real-backend integration questions, not PR2 requirements. Store does not call or depend on the engine.

## 4. Check semantic applicability, not allocation identity

**Selected: Option B.** PR1 records B plus full reconstructed `PublicInput`; admission and completion must establish that the statement still applies to trusted live context.

**Why:** equivalent allocations and unrelated snapshot updates are not changed statements. Do not silently rebind an old proof or put BLS, state loading or payload hashing on the mutator.

**Alternative: Option A:** change Gloas payload acceptance to prevent replacement. That expands scope into payload-processing policy. Pointer equality and a general generation framework were also not selected.

**Open gate:** define cheap, synchronized live evidence and account for every writer/pruner. Comparing the prepared identity with itself or trusting the accepted marker alone is insufficient. Cache eviction is not automatically a semantic change.

## 5. Follow the existing retained-block domain

**Selected:** retain the latest finalized anchor and retained unfinalized roots. Prune verified proofs and both Seen collections together; use the same eligibility domain at admission/completion.

**Why:** no historical proof-serving requirement exists. Historical block accessibility must not allow readmission, and late results must not resurrect retired records.

**Alternative:** a historical window or independent capacity eviction. These add policy or lose deduplication while targets remain eligible. Retention is not a hard memory cap during non-finality.

## 6. Allow distinct attempts; preserve first-valid storage

**Selected:** different R and prover keys may overlap for the same `(B,T)`. Each already-admitted, still-applicable attempt keeps its own verdict. Store only the first valid bare envelope; new admissions after storage IGNORE.

**Why:** the two duplicate keys do not impose exclusive ownership of `(B,T)`. Another proof finishing first is not context invalidation.

**Alternative:** one in-flight attempt per `(B,T)`. This simplifies resource handling but lets one slow attempt suppress independent validators. Distinct-attempt admission does not promise unlimited engine concurrency.

## 7. Verify once using protected provenance and non-blocking handoffs

**Selected design:** prepare on a worker → live mutator admission → engine worker → mutator completion. Do not block a worker waiting for admission. Completion must remain tied to the exact admitted input and trusted engine result.

**Why:** expensive work stays off the mutator; stale snapshots never authorize engine entry. Protected carriers prevent arbitrary envelope/boolean pairs from impersonating verified completion, but trusted orchestration must still actually run the engine.

**Alternative:** completion-only checking, synchronous worker waits, or repeating engine verification on the mutator. These respectively miss exclusion, occupy shared workers, or stall serialized processing.

## 8. Keep compatibility evidence and future exposure separate

**Selected:** retain PR18’s progressive versioned hashes with the selected list-limit semantics; defer #5729. Independent vectors/real-proof interoperability are outside PR2/PR3, but still required before interoperability sign-off.

**Why:** avoid an incidental data-model migration or scope expansion. Grandine-only expected roots are regression tests, not an independent oracle.

**Alternative:** adopt anticipated spec changes or require interoperability work inside every lifecycle PR. Neither is the selected sequence. Before live ingress, separately enforce message-size, queued-work/bytes and verifier-concurrency limits; low priority is not isolation or DDoS protection.
