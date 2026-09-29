# PR2 / D3 understanding report — execution-proof gossip and concurrent claims

**Status:** pre-implementation report; **no code changes authorized by this document**. Read with the [interview decisions](decisions.md) and [four stale-snapshot sequence diagrams](stale-snapshot-sequences.md). This reports agreements, explains why they are needed, and identifies implementation details that must be checked before opening PR2/PR3. The source of truth remains [design 0004](../../design/docs/design/0004-execution-proof-gossip-validation-pipeline.md) and its pinned consensus-specs commit `6946894b02f6e95bf5b1daf4ce39a3c3244a1e41`; update design 0004 explicitly if the proposed API/PR split changes.

## 1. Where the stack stands

- **PR1:** [Grandine #17](https://github.com/eip8025-grandine/grandine/pull/17), “Authenticate execution proof envelopes and bind payload public input,” is a **draft**. The inspected local `feature/proof-envelope-auth-binding` head was `fac019b`, four commits above PR #15 tip `e896cbd`. Its `fork_choice_store/src/store.rs` helpers authenticate a signature/envelope against a caller-supplied post-state and accepted payload, and construct a payload-bound `ExecutionProof` for the engine. It includes focused tests. At inspection, GitHub exposed no reviews or check runs and the PR description still asked for a post-fixture rerun. **Operator update:** Cargo tests and `scripts/ci/clippy.bash` were rerun after that change and passed, so the PR1 test/Clippy follow-up is **done** (operator-reported; not independently rerun here). A local `git diff --check e896cbd..HEAD` passed during this discussion. PR1 does **not** implement gossip validation or the claim protocol.
- **PR2 as originally written in design 0004:** add ordered `validate_execution_proof_gossip`, `apply_execution_proof`, Seen tracking, mutator acceptance, pruning, and settle D3. **PR3:** replace the task stub and complete controller/mutator wiring. The chosen *pre-engine claim* changes the shape of those interfaces; do not assume the original one-call `validate_execution_proof_gossip(..., verifier)` can remain unchanged.
- **Grandine today:** `fork_choice_store/src/store.rs` has Seen maps `execution_proof_roots`, `execution_proof_provers`, verified-proof map `execution_proofs`, and an accepted-payload **root set** plus `ExecutionPayloadEnvelopeCache`. `fork_choice_control/src/tasks.rs:727+` has a `ProcessExecutionProofTask` taking an `Arc<Store>` snapshot, but still returns stub IGNORE (`:752`). `fork_choice_control/src/mutator.rs:2619+` signals results but does not store bare messages (`:2641`). `controller.rs:1002+` says no ingress currently calls its spawn helper. `proof_engine/src/engine.rs` exposes a synchronous object-safe `bool` verifier; only null and mock implementations are present on this branch.

**Interpretation:** PR1 supplies the expensive authentication/binding *building blocks*. PR2/PR3 must decide **when** a worker is allowed to invoke the verifier and **who** is allowed to change the live fork-choice store.

## 2. The pinned spec rule, without Rust terminology

Let `M = signed.message`, `B = M.beacon_block_root`, `T = M.proof_type`, `V = signed.validator_index`, `R = hash_tree_root(M)`, and `K = (B,T,V)`.

`M` includes `proof_data`, `proof_type`, and `beacon_block_root` (`types/src/eip8025/containers.rs:142–147`). Two validators can sign **the same M** (same R, different signatures/indices) or submit **different proof bytes** for the same `(B,T)` (different R). `R` hashes the **bare message**, not the signed wrapper.

At the pinned `p2p-interface.md` gossip function, the order is:

1. Compute `R`; if it is already Seen, IGNORE.
2. Unknown block: IGNORE. Known block without valid post-state: REJECT.
3. Payload unavailable: IGNORE. Verified proof already stored for `(B,T)`: IGNORE. Prover `K` already Seen: IGNORE.
4. Authenticate the envelope using the block post-state and accepted payload: if invalid, REJECT **without Seen**. Bind the accepted payload and state to the proof-engine input.
5. Mark **both** R and K Seen **before** asking the engine. Definitive engine `false` is REJECT but remains Seen; `true` succeeds. Only a successful verified bare proof can later enter `execution_proofs[B][T]`.

`Seen` is **attempt memory**, not proof validity. `execution_proofs` is **success storage**. Grandine physically puts both in a Rust struct named `Store`, but that does not turn an engine-invalid proof into a verified proof or affect fork-choice voting weight. The pinned spec describes sequential logical steps; it does **not** give a Rust locking/worker protocol or define simultaneous completion ordering. Later unmerged spec PRs have changed other checks: do not borrow their check order into this pinned series.

### Four direct outcomes

| Incoming attempt | Seen R/K | Stored verified `(B,T)` | Gossip result |
| --- | --- | --- | --- |
| Bad signature or other failed authentication | No | No | REJECT |
| Authentic envelope, invalid proof (`engine=false`) | **Yes** | No | First REJECT; later replay IGNORE |
| Authentic envelope, valid proof (`engine=true`) | **Yes** | Yes if first stored valid proof | ACCEPT |
| Block/payload unavailable before authentication | No | No | IGNORE (known block without post-state is the distinct REJECT case) |

The null verifier opts out with IGNORE before this pipeline. A `false` from `NullProofEngine` must **not** be confused with a real verdict; the task checks `is_null()` first.

## 3. The Rust/Grandine concurrency model we must preserve

Think of the **mutator** as the one authorized editor of the live ledger. A worker has a read-only `Arc<Store>` snapshot: a coherent earlier view, not a lock on future changes. Grandine publishes a new snapshot with `update_store_snapshot()` (`fork_choice_control/src/mutator.rs:5371+`), but a worker holding S0 continues reading S0. `store_mut()` uses `make_mut()` (`:5381+`); simply cloning a snapshot does not give the worker authority to claim a key.

The proof task is in the low-priority work queue (`fork_choice_control/src/thread_pool.rs:282+`). The thread pool has a finite number of worker threads, but its queue is a `VecDeque` with no explicit proof-specific bound visible here (`thread_pool.rs:83+`). Hence we chose a two-stage handoff rather than blocking a pool worker on a mutator reply:

```text
Stage 1 worker: snapshot prechecks -> authenticate -> bind input
                  -> send ClaimRequest(R,K,input,context,origin); stop
Mutator:        check LIVE state -> atomically claim both keys with token t,
                  or signal IGNORE/REJECT; if granted, schedule stage 2
Stage 2 worker: engine(input) -> send Finish(t,result)
Mutator:        recheck token/context -> commit Seen; store only if valid;
                  signal ACCEPT/REJECT, or IGNORE if claim/context invalidated
```

The original worker **does not sit waiting**. The engine receives the prepared input, not the store snapshot, and returns a `bool` to the mutator via a task result. The mutator performs only short admission/finish transitions, not the potentially slow verifier call.

## 4. Why completion-only Seen cannot solve D3

**Sequential replay flaw:** with a single worker, an authenticated bad proof X returns `false`. If Seen is updated only on ACCEPT, the mutator sends REJECT but records nothing; replaying X makes the next worker authenticate and run the engine again. That contradicts the pinned rule. A special authenticated-invalid result that marks Seen at completion fixes sequential replay, but not the concurrent race.

**Old-snapshot race:** workers A and B can both read S0 with `Seen={}`, authenticate the same X or different X/Y sharing K, and enter the engine before either completes. A later mutator recheck can prevent inconsistent **writes**, but cannot refund either engine call. If X is invalid and Y valid, completion order may choose which one consumes K; that differs from admitting one authenticated attempt before verification. A single writer is not sufficient unless that writer makes the claim **before** engine entry.

**Selected linearization point:** the mutator's successful, serialized paired claim of R and K against live state. The check and registration of *both* keys must be one transition; a worker cannot reserve just R and race another on K. Pending claims gate other workers immediately even while old snapshots remain in use. This protects *same root or same prover key* engine admission on the ordinary path; it does not prevent already-paid network/decode/hash/BLS work.

## 5. Proposed state machine and ownership

This is a behavioral sketch, **not** an approved Rust API or final error type:

```text
Absent (no pending, no Seen, no stored proof)
  -- authenticate + live check + atomic claim --> Pending {R, K, token, bound_context}
Pending
  -- same R or K from another claim ---------> losing request IGNORE; Pending unchanged
  -- definitive engine false + valid context -> Seen(R,K); no verified proof; REJECT winner
  -- definitive engine true + valid context --> Seen(R,K); store bare proof if first; ACCEPT winner
  -- relevant context gone / block pruned ----> remove/void claim; IGNORE late finish
  -- confirmed worker stopped without result -> release pending; no permanent Seen
  -- elapsed clock only ----------------------> do NOT release while call may still be running
```

A fresh per-claim token (generation/task identity, whichever is simplest) is checked with both pending keys on `Finish`. If an old claim was removed and a new claim for the same keys later granted, a late `Finish(old_token)` cannot finalize the new claim. No generalized lease service or token framework is requested. Claim state is to live in `fork_choice_store::Store` alongside Seen/verified proofs, and **only the mutator** may grant/remove it. The existing sidecar-construction-started marker, accepted payload-envelope set, and gossip-attestation Seen records are useful local parallels, **not** a ready-made paired-claim implementation (see `decisions.md`).

**Spec mapping and deviation:** a pending reservation acts as the pre-engine gate while alive; a definitive engine verdict converts it to permanent Seen. The pinned pseudocode writes Seen *itself* before the engine call. This difference matters if a worker never finishes: this design will not permanently poison an authenticated key merely because an operational call got stuck. Claim/Seen must never be exposed as a verified proof. Spell this deviation out in revised design 0004 and PR descriptions.

### Two simultaneous validators, same block/type

- Same bare message R, even with different validator signatures: **second loses the pending-root claim** and gets IGNORE without engine work.
- Different messages R and validator keys K: **both may claim** while no verified `(B,T)` proof exists (agreed Q13). An invalid A does not prevent valid B from supplying a verified proof. An additional per-`(B,T)` in-flight gate would change the pinned rule.
- Both were admitted before either proved valid: if A stores a valid proof while B runs, B still gets its own **engine-based verdict** for peer scoring—ACCEPT if valid, REJECT if invalid (agreed Q14)—and commits its authenticated Seen keys, provided its context/token remain valid. The first stored verified proof is preserved; B never overwrites it. A **new** request after storage is IGNORE before engine work. This is a documented *local concurrent ordering policy*, not an instruction present in the sequential spec.

## 6. What "stale snapshot" does and does not mean

The diagrams in [stale-snapshot-sequences.md](stale-snapshot-sequences.md) show the distinct windows. A snapshot may be old the instant it is handed out. **Do not compare a global snapshot version and reject everything merely because it changed.** Decide using relevant live context and ownership at two serialized points:

| Window | Decision | Expected treatment |
| --- | --- | --- |
| Stage 1 sees old S0; another claim/Seen/stored proof wins before ClaimRequest | Live mutator admission | IGNORE stale competitor; no engine. Keep pinned early-check precedence. |
| S0 contains accepted payload, but live payload is now missing/different before grant | Live mutator admission | IGNORE rather than engine verification against an unrelated old binding; no claim. |
| Known block has no valid post-state at relevant validation check | Pinned D2 check | REJECT (not a blanket stale IGNORE). |
| Context changes after grant but before stage 2 starts | Mutator may invalidate token; finish rechecks | A queued task **might still run**; do not commit on its late result. A pre-start cancellation optimization is not selected. |
| Block pruned or relevant state/payload changes while engine runs / while Finish waits | Live mutator finish | IGNORE; no Seen or verified proof insertion; no resurrection. |
| Unrelated tick or snapshot publication, binding and token unchanged | Live mutator finish | Still honor the result; a newer snapshot alone is not a stale proof. |
| Another distinct validator stores a valid proof while an earlier granted B runs | Live mutator finish | Honor B's engine verdict and Seen; do not overwrite stored proof (Q14). |

Stage 1's authentication/binding must use one coherent block-state/payload pair; `Store::validate_execution_proof_gossip` cannot assume the spec's payload-envelope dictionary exists in Grandine. `Store.payloads` is a set; fetch the **accepted** envelope from `ExecutionPayloadEnvelopeCache` (D1); a missing cache entry gives IGNORE. A known block without usable post-state is the separately pinned D2 REJECT. At claim and finish, the mutator must compare **the relevant binding context**, not simply S0 versus S1 publication counters. Exactly how to identify a matching post-state and accepted payload without expensive reauthentication on the mutator is **open engineering design**. Proof storage by another validator is explicitly not a binding change for already admitted B.

**Correction to a common mental picture:** the engine does not update the store or receive a snapshot result. It sees immutable reconstructed proof input. A late `true` gives the mutator *evidence of validity under the old input*, not permission to mutate live state; a token and context recheck decide whether it is applicable now. `IGNORE` for a locally stale/pruned claim is not an assertion that the cryptographic proof is false and should not penalize the peer.

## 7. Lifecycle, safety, and resource limits

**Retention:** Grandine already calls `Store::prune_after_finalization` (`store.rs:5473+`) and prunes payload data against retained block roots (`:5487+`). PR2 should extend that sweep to both proof Seen maps, verified proofs, and pending claims using one consistent retained-block policy, with special attention to the last finalized root. A late finish cannot reinsert an entry for a block that was pruned. The precise retention predicate and outstanding-claim handling need tests. Proofs need not persist across restart in this series; do not claim restart recovery is implemented.

**Non-completion:** the current synchronous `ProofVerifier` returns `bool` (`true` valid, `false` definitively invalid) and has no typed unavailable/timeout result. We agreed **not** to expand that API in this stack. Do not map a real verifier's transient infrastructure failure to definitive `false` later. A timer may diagnose a stuck worker but must **not** release its claim while its call may still run; token fencing alone cannot prevent two overlapping engine calls. Confirmed cessation, block pruning, or process shutdown are different events. How a panic, enqueue failure, shutdown, or indefinitely hung call reliably releases/retains a claim is **not yet specified**; this must be resolved to ship a live integration, not left as an accidental leak or false DDoS guarantee.

**Capacity:** proof data is bounded (the signed-envelope pre-decode cap is declared in `types/src/eip8025/consts.rs`), but per-key claims cannot defend against many distinct validly signed keys, duplicated BLS checks on stale snapshots, large-message decode/hash costs, or an unbounded low-priority task queue. A synchronous verifier can occupy shared worker threads. The user chose to defer broad adversarial DDoS testing and real-engine failure injection until an actual verifier is integrated, **not** to declare the service safe to expose without admission, queue, pending-claim, and verifier-concurrency limits. No active proof ingress is present in this skeleton; treat those limits as a prerequisite to enabling exposure, with an explicit gate in the PR plan. Avoid claiming absolute DDoS prevention.

**Complexity sketch, not a benchmark:** with hash maps/sets, paired root/key lookup and a token ownership check should be expected-average O(1) per claim/finish; root hashing is O(message/proof byte length), BLS and proof verification have engine-specific cost, and pruning scans retained entries. Store copy-on-write snapshots and map cloning may alter real costs; benchmark only when an actual workload/verifier exists. No queue or memory bound follows from O(1) lookup.

## 8. Minimal test contract (before a real engine)

Use a deterministic mock verifier and barriers/latches to force interleavings; assert **engine-call counts**, network/API verdicts, and all Seen/pending/stored maps. At minimum:

1. Invalid signature/unsupported envelope -> REJECT; no claim, no Seen; a valid later attempt with that claimed validator is not poisoned.
2. First authenticated proof, engine `false` -> REJECT; both Seen; no verified proof. Sequential same-message replay and new bytes from same V -> IGNORE, engine count stays one.
3. Two same-root or same-K workers on S0 -> at most one **admitted** engine call; losing claim IGNORE. Different validator but same bare message root also loses; different roots/validators may both be admitted.
4. A invalid / B valid for same `(B,T)`, different validators: B can be stored. A valid / B invalid: A stored, B's earlier admitted proof REJECTED. Both valid: first stored proof retained, both earlier admitted verdicts ACCEPT. A new attempt after storage IGNORE.
5. Context changes before claim -> no engine, appropriate IGNORE or pinned missing-post-state REJECT. Context disappears after claim, including during engine and after completion but before Finish -> IGNORE and no Seen/verified write. Unrelated snapshot publication -> preserve verdict.
6. Finalization prunes all four kinds of proof bookkeeping consistently. Late old-token finish after pruning or a new generation -> IGNORE; cannot resurrect data or complete the new claim.
7. Null verifier -> IGNORE with no claim and no engine call. Initial payload-cache miss -> IGNORE; compare D1/D2 behavior.

Tests of real-verifier cancellation, malicious ingress rates, saturating queues and runtime budgets are **deferred**, but the deliberate concurrency and stale-context invariants above are **not** deferred merely because the engine is mocked.

## 9. Remaining design work and proposed PR boundary

These are **open engineering details**, not requests to reopen the agreed behavioral answers:

1. Identify exactly what stage 1 carries to let the mutator cheaply compare the authenticated state/accepted payload with its live store. Preserve the pinned early IGNORE/REJECT check order; do not rerun expensive BLS on the mutator. Keep the input bound to the *same accepted payload*, not another envelope at the same block root.
2. Define the smallest pending-claim representation, atomic paired-key insertion/removal, token generation/non-reuse, finish ownership test, and memory/pruning behavior. Decide what happens if task scheduling fails after a claim, a worker panics, or the process stops. Confirm no stale completion can reinsert pruned keys.
3. Spell out the precise messaging/origin signalling and test for each outcome, including REJECT for authenticated engine-invalid first attempts, IGNORE for pending losers and stale context, and ACCEPT for already admitted valid proofs without overwriting stored data.
4. **Revise design 0004's PR split before coding.** A plausible *proposal*, not yet approved: PR2 defines ordered stage-1 validation, live claim/finish/pruning store operations, and unit/race tests for those transitions; PR3 adds mutator claim messages, scheduling the non-blocking stage-2 task, result signalling, and integration tests. Alternatively, a thin claim-message seam might need to appear in PR2 to test the end-to-end concurrency invariant. Do not advertise PR2 as implementing the full `validate_execution_proof_gossip(..., verifier)` method if the verifier is intentionally in stage 2. Pin the updated design and keep the final diff reviewable.
5. Define a **no-live-ingress gate** until finite admission/queue/verifier resource bounds exist. Existing `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` must be enforced at the relevant transport decode boundary when ingress is added; the constant alone does not enforce a network limit.

### Decision summary

- **Settled:** pre-engine two-key claim, non-blocking two-stage handoff, pending in `Store` owned by mutator, smallest unique token, no timeout-only release, pruning in PR2, independent different-validator/different-root claims, engine-based verdicts for already admitted tasks, first valid stored proof retained, live relevant-context checks at claim and finish.
- **Not settled / not implemented:** exact context identity, lifecycle on worker failure/panic/enqueue failure, capacity settings and ingress gate details, final Rust APIs and PR2/PR3 seam. These can be decided from this report and code evidence before any implementation; they are not evidence that the existing skeleton already prevents DDoS.
