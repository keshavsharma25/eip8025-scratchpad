# Design A: best-effort proof context after ordinary EL notification

**Status:** detailed design for operator review; no Rust implementation authorized or performed by this document.
**Direction selected in discussion:** Design A, with strict failure isolation and limited synchronous in-memory capture.
**Primary invariant:** proof-context bookkeeping must not change the validity or ordinary outcome of an execution payload envelope.
**Tradeoff:** synchronous capture adds work after `notify_new_payload` and before returning Accept. It is not latency-neutral and does not meet the earlier requirement of no proof computation delaying acceptance. That requirement is explicitly relaxed for this design; no performance approval is implied.

## 1. Executive overview

Keep Grandine's ordinary payload validation and state selection. Once normal checks pass, call its existing execution-engine interface. Only after successful submission, try to construct proof public input from the already available **block bid and payload envelope**, not from additional state reconstruction.

```text
Existing ordinary payload checks and state selection
    │ ordinary errors/delays keep their original behavior
    ▼
notify_new_payload(B, payload, params, None)
    │ existing submission error keeps its original behavior
    ▼
Best-effort proof-context capture (validation task, not mutator)
    ├─ success → optional Q = Some(Q)
    └─ failure → bounded diagnostic; optional Q = None
    ▼
Return ordinary Accept with envelope + optional evidence
    ▼
Serialized application
    ├─ existing payload markers, deadline bookkeeping and cache selection
    └─ synchronize proof evidence: install Q or invalidate the previous Q
    ▼
Existing snapshot publication, origin handling and ordinary follow-ups
```

There is no new proof task, queue, completion message, selection generation, retry loop, proof backend call, or deferred-context protocol. This is the main simplicity advantage over Design B.

The core outcome must be identical whether capture returns Some or None. The remaining synchronous request construction/hashing cost is real; prohibit additional I/O/replay and measure the cost rather than describing it as non-intrusive in every sense.

## 2. Current source versus historical CP2 work

Fresh local reads for this proposal show:

- Branch: `feature/proof-store-lifecycle`.
- HEAD: `80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4`.
- `store.rs` currently returns raw `ExecutionPayloadEnvelopeAction` and applies an `Arc<SignedExecutionPayloadEnvelope>`.
- The ordinary lazy state closure uses `state_cache.before_or_at_slot`, then `ChainLink::state` as fallback.
- The ordinary validation tail calls `notify_new_payload` and directly returns raw Accept.
- `get_execution_proof` currently contains the state-based binding implementation inline; no `get_execution_proof_public_input` helper was present in this fresh read.
- `PreparedExecutionProof` and B + complete Q identity remain present; proof context loading uses `try_state_by_block_root`.

These observations differ from the earlier coupled CP2 working-tree implementation. Do not assume its protected carrier, map, helper or tests are currently restored. A broad Git status command timed out; no staged/unstaged cleanliness or checkpoint-preservation claim is made. The branch/HEAD commands used a per-command `safe.directory` setting, not a persistent Git configuration change.

The [original alternatives document](design.md) describes that earlier coupled working tree and the choices considered. Its recommendation of B and rejection of A under stronger latency isolation are superseded **for the selected direction** by this document. Historical evidence remains historical.

Before implementation, establish the actual staged/unstaged diff and agreed PR17/upstream baseline again. No checkout, restoration, commit or rebase is part of this design work.

## 3. Requirements and non-goals

### Hard requirements

1. No proof-only state selection, exact-state reconstruction, request construction, hashing, BLS, disk access or engine verification before ordinary `notify_new_payload`.
2. No change to ordinary validation order or its Reject/Ignore/Delay/error semantics solely to obtain proof evidence.
3. A capture `Err` is logged and converted to **absent evidence**, never returned through ordinary payload error handling.
4. No capture-dependent failure at carrier construction or application. Optional evidence is an ordinary valid state of the carrier.
5. No additional state/cache/persistence reads or replay during synchronous capture. Reuse the target block bid and envelope already held by ordinary validation.
6. No hashing, proof-byte copying, BLS, request reconstruction or engine call on the serialized mutator.
7. Full Q remains the content identity used for proof applicability. Do not substitute execution hash, pointer, snapshot revision or payload slot.
8. Every ordinary selected-payload replacement also replaces or removes its proof evidence. Absence must never preserve the old Q.
9. Strict trusted-state reconstruction/authentication stays on the proof-preparation path.
10. No `proof_engine` edits, calls or new Store engine dependency in PR2.

### Explicitly accepted limitation

After successful notification, capture performs synchronous in-memory work. “Bounded” means bounded by admitted input sizes/configuration and constructor limits, not small or constant-time. It does not prove that processing deadlines or worker throughput are unaffected.

Failure isolation here covers normal returned construction errors. It is not a guarantee against process OOM, allocator abort, unexpected panic or severe CPU exhaustion. Do not use panic-catching to pretend otherwise. Inspect introduced code for unnecessary allocations and unchecked assumptions.

### Outside scope

No changes to payload-validity policy, proof generation, fork-choice weights/head policy, beacon-state transition rules, ingress/network topics, backend behavior, containers or request-root convention. No new asynchronous capture machinery. Physical proof-map/Seen/verified-proof pruning is deferred to a separate PR **after this stack**; existing ordinary cleanup remains intact.

## 4. Source-grounded architecture and ownership

### Representative existing locations

- [`store.rs`](../../grandine/fork_choice_store/src/store.rs): `validate_execution_payload_envelope`, `validate_execution_payload_envelope_with_state`, `get_execution_proof`, `apply_execution_payload_envelope`.
- [`misc.rs`](../../grandine/fork_choice_store/src/misc.rs): existing public raw `ExecutionPayloadEnvelopeAction`.
- [`prepared_execution_proof.rs`](../../grandine/fork_choice_store/src/prepared_execution_proof.rs): protected carrier, private constructor, immutable scalar accessors, strict proof context loading.
- [`execution_payload_envelope_cache.rs`](../../grandine/fork_choice_store/src/execution_payload_envelope_cache.rs): B-keyed overwrite and independent byte eviction.
- [`execution_engine.rs`](../../grandine/execution_engine/src/execution_engine.rs): `notify_new_payload` consumes payload and params and returns `Result<()>`.
- [`container_impls.rs`](../../grandine/types/src/eip8025/container_impls.rs): `NewPayloadRequest::new` borrows inputs but constructs owned fields, and `zkboost_compatible_root` hashes the selected statement.
- [`messages.rs`](../../grandine/fork_choice_control/src/messages.rs), [`tasks.rs`](../../grandine/fork_choice_control/src/tasks.rs), [`mutator.rs`](../../grandine/fork_choice_control/src/mutator.rs): existing result transport, validation task and serialized ordinary apply/follow-ups.

Keep the capture/binder helpers private in `store.rs` initially, next to the existing binder. Keep action types in the established actions area in `misc.rs`, reexported through `lib.rs`. A private Store-constructed accepted carrier can remain beside Store validation code, following the existing preparation carrier's privacy principles. Do not add special path/module wiring to avoid a straightforward private helper.

### The ownership constraint that matters

`notify_new_payload` consumes the assembled payload and `Option<ExecutionPayloadParams<P>>`. Therefore, after notification, the exact owned arguments are no longer available to borrow.

Do **not** solve this by:

- cloning extra payload/parameter objects before notification solely for proofs;
- changing the ExecutionEngine trait to return a proof receipt;
- computing the request root before notification;
- keeping another large proof-specific owned buffer alive across the call.

The validator still owns the envelope Arc and a borrowed block bid through its block Arc. After notification, reconstruct equivalent request inputs from those same immutable sources. This deliberately repeats some assembly after the call. It is simpler and keeps the existing engine API intact; its cost must be acknowledged.

## 5. Statement definition and equality argument

Let B be the beacon root. Define Q as:

```text
Q = {
  new_payload_request_root: zkboost_compatible_root(NewPayloadRequest),
  successful_validation: true,
  chain_id: configured deposit chain ID,
  schema_id: STATELESS_INPUT_SCHEMA_ID,
}
```

`successful_validation = true` is the assertion the proof is expected to establish. Installing this statement does **not** assert that the EL has returned VALID. `notify_new_payload(..., None)` is submission, not a final execution verdict; Store acceptance also is not EL VALID.

### Capture and the ordinary submitted request

The ordinary validator already selects:

- execution payload: `envelope.message.payload`;
- versioned hashes: KZG commitments from the **target block's bid**, mapped with the existing helper;
- parent beacon root: `state.latest_block_header().parent_root`, which the existing ordinary check requires to equal `envelope.message.parent_beacon_block_root`;
- execution requests: `envelope.message.execution_requests`.

The capture path must use exactly those sources, in the same order and with the same conversions. Ordinary EL submission and capture both operate on the same retained envelope/block values. No state is consulted by capture, so an unrelated or poisoned state's latest bid cannot silently become the source of live Q.

The parent-root equality is already enforced before notification by `ExecutionPayloadParentBeaconBlockRootMismatch`; capture does not add that validation or re-read state. This establishes field equality to the submitted request by data flow and the existing ordinary check. A notifier spy must additionally compare the full reconstructed request, not just B or its declared execution hash. If that ordinary check/source changes upstream, revisit the argument rather than assuming matching variable names establish equivalence.

### Capture and PR17's proof binder

PR17's `get_execution_proof` derives versioned hashes from the trusted target post-state's latest bid. Preserve that spec-facing trust boundary. Factor only its pure construction into a shared helper.

For a healthy validated target, prove that the trusted post-state's latest bid commitments match the target block bid used for ordinary EL submission. Combined with the same envelope/configuration and shared root helper, both paths produce equal Q.

If locally inconsistent state produces a mismatch, strict preparation must fail or live proof applicability must decline. It must **not** be repaired by changing ordinary acceptance, overwriting captured Q with the preparation's Q, or accepting whichever request root is convenient.

The selected root is the PR #20 **zkBoost-compatible** root. Canonical progressive SSZ hashing/encoding remains untouched. A root-convention change is not a cleanup in this redesign.

## 6. Interface proposal

All signatures below are proposed Rust shapes, not claims that the APIs already exist. Final names/imports follow repository conventions during implementation.

### 6.1 Pure shared PublicInput builder

```rust
// Private Store helper. No state loading, storage, logging or Store mutation.
fn execution_proof_public_input_from_parts(
    &self,
    payload_envelope: &ExecutionPayloadEnvelope<P>,
    versioned_hashes: impl Iterator<Item = H256>,
) -> Result<PublicInput>;
```

It builds the Gloas payload/params with the existing `NewPayloadRequest::new`, computes `zkboost_compatible_root`, and fills the other Q fields from immutable configuration/constants. The iterator is consumed only when this helper is called. It hides the request constructor/root convention behind one narrow internal interface, without adding a new generic subsystem.

Caller 1: PR17's `get_execution_proof` checks Gloas state and derives hashes from its trusted state bid, then invokes this helper. It constructs `ExecutionProof` only after the fallible Q construction succeeds, avoiding a new proof-byte clone before a possible binding failure.

Caller 2: ordinary validation invokes the helper **after** notification using hashes derived from its already selected block bid, handling any error locally.

If keeping a state-facing `get_execution_proof_public_input(state, envelope)` wrapper improves readability/reuse, it may delegate to this helper. It must not be called by ordinary capture, and must not independently load state. The pure helper is the shared implementation; the state-based and block-bid-based source selection remain visibly distinct.

### 6.2 Ordinary accepted carrier with optional proof evidence

```rust
pub struct ValidatedExecutionPayloadEnvelope<P: Preset> {
    envelope: Arc<SignedExecutionPayloadEnvelope<P>>,
    proof_public_input: Option<PublicInput>,
}

impl<P: Preset> ValidatedExecutionPayloadEnvelope<P> {
    // Private to Store validation. Both Some and None construct successfully.
    fn from_validation(
        envelope: Arc<SignedExecutionPayloadEnvelope<P>>,
        proof_public_input: Option<PublicInput>,
    ) -> Self;

    pub fn beacon_block_root(&self) -> H256;
    pub fn slot(&self) -> Slot;
    pub fn builder_index(&self) -> BuilderIndex;
    pub fn payload_block_hash(&self) -> ExecutionBlockHash;
}
```

“Validated” means ordinary Store validation/submission succeeded, not that context exists or EL declared VALID. Unlike the historical CP2 carrier, this type does **not** require `ExecutionProofContextIdentity`.

B is read from `envelope.block_root()`; optional Q does not participate in selecting the root or determining whether the carrier is valid. Fields and constructor are private; immutable scalar accessors suffice for Control. Do not expose envelope access that allows callers to mutate its cached-root cells or replace envelope/evidence independently. No arbitrary public `Some(Q)` constructor.

The carrier is retained for **pairing**, not to smuggle proof validation into ordinary acceptance. It makes it difficult for Control to apply an envelope from one result with evidence from another. A carrier with None is just as acceptable to the ordinary pipeline as one with Some.

### 6.3 Keep raw/custom-state results distinct

Retain public `ExecutionPayloadEnvelopeAction<P>` for queries/back-sync/custom-state validation. A caller-supplied state or block callback must not mint trusted live proof evidence.

For the actual Store validation task, propose:

```rust
pub enum ValidatedExecutionPayloadEnvelopeAction<P: Preset> {
    Accept(ValidatedExecutionPayloadEnvelope<P>),
    Ignore(Publishable),
    DelayUntilBeaconBlock(Arc<SignedExecutionPayloadEnvelope<P>>, H256),
    DelayUntilState(Arc<SignedExecutionPayloadEnvelope<P>>, H256),
    DelayUntilData(
        Arc<SignedExecutionPayloadEnvelope<P>>,
        Arc<SignedBeaconBlock<P>>,
    ),
}
```

Only the Accept payload differs. Ignore/Delay variants preserve existing values, queueing and retry semantics. This reuses the earlier CP2 boundary shape, but removes mandatory proof context.

A private common validation implementation can return an internal enum whose Accept holds `(envelope, Option<Q>)`. The ordinary Store wrapper enables capture; the public custom-state wrapper disables it and converts to the unchanged raw action. Conversion to the production carrier accepts None without error. No missing-evidence error may be manufactured during conversion.

A private boolean capture mode is adequate for two internal callers; do not add a public mode knob or alternate trusted constructor. Ordinary checks must be shared, not duplicated into two diverging pipelines.

### 6.4 Store evidence and application

```rust
// Existing persistent-map conventions; no extra selection generation.
execution_proof_contexts: HashMap<H256, PublicInput>

pub fn apply_execution_payload_envelope(
    &mut self,
    validated: ValidatedExecutionPayloadEnvelope<P>,
    is_before_deadline: bool,
);

pub(crate) fn execution_proof_context_is_applicable(
    &self,
    identity: &ExecutionProofContextIdentity,
) -> bool;
```

Application consumes the paired envelope/evidence. It performs the existing marker/timely/accepted-tuple/cache updates and synchronizes proof authority within the same exclusive Store mutation:

```text
if B is eligible AND evidence is Some(Q): contexts[B] = Q
otherwise:                              remove contexts[B]
```

There is no fallible proof validation during apply. Existing ordinary marker/cache behavior applies even for an ineligible proof target; the eligibility check governs only proof authority.

The cheap applicability predicate is:

```text
eligible(B)
AND ordinary accepted-payload marker contains B
AND contexts[B] == prepared.full_Q
```

It performs fixed-size comparisons/map lookups, not reconstruction. Cache byte residency, engine verdict, pointer equality and verified-proof occupancy are not statement identity. Verified occupancy and both Seen collections retain their separate admission roles.

## 7. Validation control flow: exact error boundary

Illustrative common validation tail:

```rust
// Ordinary checks and ordinary state selection are unchanged above this point.
let versioned_hashes = bid.blob_kzg_commitments.iter().copied()
    .map(misc::kzg_commitment_to_versioned_hash).collect();
let params = Some(ExecutionPayloadParams::Gloas {
    versioned_hashes,
    parent_beacon_block_root,
    execution_requests: envelope.message.execution_requests.clone(),
});
let payload = envelope.message.payload.clone().into();
execution_engine.notify_new_payload(beacon_block_root, payload, params, None)?;

// Everything newly proof-specific begins after successful submission.
let proof_public_input = if capture_proof_context {
    match self.execution_proof_public_input_from_parts(
        &envelope.message,
        bid.blob_kzg_commitments.iter().copied()
            .map(misc::kzg_commitment_to_versioned_hash),
    ) {
        Ok(q) => Some(q),
        Err(error) => {
            // Existing logging conventions; bounded context, no full payload dump.
            log_context_capture_failure(beacon_block_root, &error);
            None
        }
    }
} else {
    None
};

// No `?`, rejection or missing-context error after capture.
Ok(InternalValidation::Accept { envelope, proof_public_input })
```

`log_context_capture_failure` and `InternalValidation` are illustrative names, not assumed repository APIs. Logging should follow existing `debug_with_peers!` / warning conventions after review; do not create a logging abstraction just for this snippet.

In particular, do not use `?` on capture, `expect` on optional evidence, or `return Ignore(false)` on a capture error. Ordinary engine submission errors still use the existing `?`; this design does not relabel or swallow them.

## 8. Why no separate selection witness is needed here

Design B needed a witness because context publication arrived in a separate later completion. Design A has **one task result containing the exact envelope and optional Q**, applied together.

Consider out-of-order ordinary validation results:

```text
result A = (envelope A, Some(QA))
result B = (envelope B, None)

apply A → selected A, evidence QA
apply B → selected B, evidence absent
```

Reverse application order selects A with QA, following the ordinary existing last-application cache behavior. The redesign must not introduce first-wins or reject an ordinary late result simply because its proof context is old.

There is no independent late “publish QA” message that can reinstall evidence after B. Existing snapshot persistence means old snapshots may still contain QA; live admission/completion recheck the current Store.

Other cases:

- **A(Some QA) → B(Some QB):** selected B has QB; prepared QA declines if QA differs.
- **A(Some QA) → B(None):** remove QA immediately; old prepared work becomes locally inapplicable.
- **A → B → A:** current full Q determines applicability; no separate context-publication ABA hazard. Reuse of equal statement identity is intentional; existing Seen/storage rules still apply.
- **Equivalent allocation with successful capture:** equal Q preserves applicability; no pointer identity.
- **Equivalent allocation with capture failure:** old Q is removed. Cannot prove equivalence cheaply at apply, so temporarily lose proof availability rather than retain possibly stale authority.
- **Byte eviction only:** installed Q may survive. No new selection occurred; a previously prepared proof can remain applicable.
- **Target retired:** eligibility fails at admission/completion regardless of stale resident evidence; late ordinary payload application follows ordinary semantics but cannot revive proof authority.

Every writer of selected payload bytes/markers must be inventoried before coding. If a new writer bypasses the paired application path, it must clear evidence or supply properly paired evidence—not assume the protected task carrier covers all writes.

## 9. PR17-origin code changes, explicitly

PR17 means Grandine repository PR #17. The changes land in PR2, but **may modify code that PR17 introduced in `store.rs`**.

Expected changes:

1. Extract the inline pure request/PublicInput construction from `get_execution_proof` into the private parts-based helper.
2. Keep `get_execution_proof`'s signature, Gloas-state guard, state-derived commitment selection, root convention and returned proof fields.
3. Keep fallible binding ahead of proof-data cloning, preserving the current allocation/failure ordering where possible.
4. Retain strict `try_state_by_block_root` on proof context loading. Do not call it merely to capture evidence for ordinary acceptance.
5. Add PR2 live applicability integration to preparation where agreed, without changing early precheck/authentication precedence or consuming Seen. Preparation can locally decline if live evidence is absent/mismatched; it must never synthesize evidence from itself.

The first four preserve PR17's proof trust model while changing implementation factoring. The fifth is PR2 lifecycle integration, not a claim that preparation alone authorizes engine work. Tests must distinguish existing PR17 guarantees from added live evidence checks.

No promise is made that every PR17-origin line remains unchanged. No amendment/rewrite of PR17's Git history is required by this design.

## 10. Function-by-function implementation map

### Store validation and actions

- `validate_execution_payload_envelope`: keep the existing lazy before-or-at-slot/fallback state closure; production result becomes the optional-evidence carrier action. Do not introduce proof-only exact-state lookup.
- `validate_execution_payload_envelope_with_state`: retain its existing public callbacks, raw result contract and ordinary behavior. Capture disabled; no new authority.
- Private validation implementation: share ordinary code and append the best-effort capture tail after successful notification. No duplication of the large validation body.
- `ValidatedExecutionPayloadEnvelope`: private ordinary validated envelope + optional Q; B accessor from envelope.
- Raw/production action conversions: delays/ignores identical; production Accept accepts missing evidence.
- `get_execution_proof`: preserve state-based binding, delegate pure construction.
- New private parts helper: request construction and selected hash only; no storage/state access/logging.

### Store state and lifecycle

- `Store::new`: empty proof-context map.
- `apply_execution_payload_envelope`: ordinary application plus paired Some-install/None-invalidation. No new ordinary rejection.
- Eligibility helper: current finalized anchor or current unfinalized locations, as agreed in PR2, independently of physical proof-map residency.
- Applicability helper: accepted marker + eligible B + full Q equality.
- Proof preparation/admission/completion: consume/read authoritative current evidence; retain authentication/provenance/paired Seen/first-valid rules. No engine scheduling in Store.
- Finalization: no proof-specific bulk pruner added in this stack. Existing ordinary pruning is not removed.

### Control transport

- `ExecutionPayloadEnvelopeTask` and `MutatorMessage::ExecutionPayloadEnvelope`: transport the production optional-evidence action without inspecting/reconstructing Q.
- `handle_execution_payload_envelope`: change Accept carrier plumbing, but preserve Ignore/Delay/Err routing, metrics, origin replies, deadlines and retry queues.
- `accept_execution_payload_envelope`: read ordinary scalar accessors and pass the whole carrier to Store. Preserve existing apply → snapshot → ordinary fork-choice notification/follow-up ordering.
- Diagnostic test helper changes only if needed to observe ordinary outcomes; no fixture or protocol validation weakening.

PR2's payload-carrier transport adjustment is distinct from PR3's execution-proof engine task/message integration. Do not use this redesign to pull proof scheduling into PR2.

## 11. Local faults, diagnostics and practical limits

Capture can fail on request-binding phase/representation/bounds errors. Healthy Gloas import already narrows some cases; do not invent an expected production failure rate. Built-in commitment schedules inspected previously fit the request bound, but arbitrary custom configurations and future schedules need tests.

A failure log should identify B and a compact local construction cause. It must not claim the peer payload is invalid, dump proof/payload/state contents, or trigger ordinary import retry. Expected absent evidence when capture is intentionally disabled should not produce a warning. Retired/inapplicable proof work normally needs low-level diagnostics rather than warning spam. Review existing rate-limiting/aggregation support before adding any new mechanism.

Installing Q means the submitted ordinary statement is available for later comparison. It does not certify EL validity, proof verification, or state integrity. Capture should not mutate shared SSZ caches as a hidden way of validating state, and must not compare a cached declared state root as a shortcut for trusted context.

Performance characteristics:

- Additional validation-task work is O(request hashing/construction size), including commitment mapping and payload/request cloning or structural sharing according to existing types.
- Data-dependent cost includes payload transaction contents; it is not just O(number of blob commitments).
- Mutator work adds a map replacement/removal and fixed-size Q move/copy, subject to existing persistent-collection copy-on-write costs; no absolute constant-latency guarantee.
- Retained evidence costs O(number of recorded historical B) until the follow-up cleanup policy. Ordinary byte eviction does not bound proof-map or old-snapshot memory.

If measurement shows unacceptable worker latency, allocation or deadline effects, revisit asynchronous B or demand-triggered C. Do not silently claim A meets the former stronger latency requirement or reintroduce stricter ordinary state selection to repair capture trust.

## 12. Pruning and stack scope

Physical pruning of proof-context entries, verified proofs and paired Seen bookkeeping is deferred to a separate PR after this stack. Existing ordinary marker/cache/storage pruning continues normally.

Keep in this stack:

- current eligible-domain checks at preparation/admission/completion;
- clearing authority when the selected envelope is replaced without evidence;
- refusing authority for retired targets, even if old map entries survive;
- no Seen rollback on local failures after admission.

Deleting/replacing Q during selection is correctness invalidation, not the postponed garbage-collection project. Permanent Seen memory may accumulate; retired entries do not become an excuse to weaken eligibility. Track resource cleanup as a real follow-up before unrestricted long-lived deployment. The old CP6 pruning checkpoint and dependent CP7 plan must be revised explicitly, not silently marked complete.

## 13. Test strategy

### A. Verify ordinary behavior independently of capture

Use the freshly agreed ordinary baseline. For capture success, forced returned failure and disabled capture, compare:

- notify count, full payload and full params;
- ordinary Accept/Ignore/Delay/error outcome;
- accepted/timely markers, accepted tuples and cache content;
- snapshot publication and required origin response/follow-up behavior.

The result shape differs, but ordinary observables must not depend on Some versus None. Engine submission failure must retain ordinary error handling and skip capture entirely. Normal invalid signatures/DA/block/bid/state failures must retain prior ordering and outcomes; proof capture should never run for them.

### B. Enforce operation order and scope

Use test-only spies/recording hooks following existing fixture conventions, not a new production dependency-injection framework solely to force errors.

Record sequence: ordinary checks → notify → capture → Accept. Assert no proof-only exact-state load, storage read, replay or request hash before notify; no capture reconstruction on the mutator. A blocked capture demonstrates the **accepted limitation** that return of Accept waits for it—do not write a false test asserting asynchronous completion.

Cover realistic constructor errors where feasible. If a valid ordinary fixture cannot naturally produce a capture error, use a narrowly scoped test-only failure seam and document that it verifies control-flow isolation, not a reachable network case. Do not manufacture an invalid ordinary envelope and call its acceptance a success criterion.

### C. Statement equality and PR17 compatibility

- Healthy resident, cached, persisted, reconstructed and next-slot-only-cache ordinary cases preserve baseline behavior.
- Notifier spy's actual submitted request equals reconstructed capture inputs in every field.
- Trusted PR17 preparation Q equals captured Q on healthy fixtures.
- Each Q field independently mismatching declines live applicability.
- Preserve the PR #20 compatible-root regression against accidental canonical-root substitution.
- Preserve constructor bounds and pre-Gloas binder errors; no proof-data clone needed before fallible binder completion.
- Poisoned local state remains untrusted on the proof path. Ordinary behavior matches its baseline; do not demand that every poisoned state is accepted or alter ordinary checks to obtain that result.

### D. Pairing and replacement

Test Some→Some, Some→None, None→Some, None→None, equivalent content, changed content, reverse-order task application and A→B→A. Confirm selected bytes and installed/absent Q correspond after every apply. Old snapshots cannot authorize live admission after replacement. Byte eviction alone preserves installed evidence. Retired-target applicability fails without relying on map cleanup. No physical-pruning test is required to pretend the deferred work exists.

### E. Lifecycle and boundary regressions

Retain PR17 preparation/authentication/ordering/private-carrier tests. Add missing-evidence local declines without premature Seen mutation; preserve paired permanent Seen, exact completion provenance and first-valid storage behavior in their own checkpoints. Custom-state validation cannot mint production authority. Some and None accepted carriers have the same ordinary Control response behavior.

## 14. Evidence, acceptance and implementation sequence

The operator's prior **57 passing focused Store tests** and reported successful Clippy concern the previous current-working-tree implementation, including unstaged tests/repairs. They do not execute this design and do not certify the now-observed baseline or staged snapshot. VM Cargo was historically blocked before compilation by missing pinned offline source; no repair/fetch is authorized. The three unrelated Gloas failures remain set aside; do not investigate them or alter fixtures/validation in this work.

Suggested bounded implementation steps after approval:

1. Establish/checkpoint current Git/index/working-tree state, agreed PR17 base and upstream/fixture baseline. Preserve all operator changes. No commit/rebase without explicit approval.
2. Extract the pure shared builder while preserving PR17 semantics; add equivalence/root/allocation-order regressions.
3. Add optional-evidence accepted carrier and common validation result without changing ordinary checks; append post-notification best-effort capture.
4. Add paired Store evidence install/invalidation and cheap applicability, including the failure/replacement matrix.
5. Adapt existing payload result transport and verify all ordinary responses/follow-ups. No proof-engine orchestration.
6. Reconcile operative PR2 checkpoints/decisions, defer CP6 cleanup explicitly, and continue lifecycle checkpoints only after acceptance.
7. Run repository-appropriate fmt, focused and affected Store/Control tests, CI Clippy; distinguish host and VM evidence and record exits/skips. Measure critical-path overhead before asserting it is acceptable.
8. Obtain independent correctness/architecture/readability review when delegation is authorized. Parent independently validates any delegated findings/changes.

Approval gates before coding:

- [ ] Accept synchronous in-memory capture after submission and its possible latency cost; preserve the no-proof-computation-before-notification boundary.
- [ ] Accept source-based reconstruction from the already held block bid/envelope, with explicit full-field EL/PR17 equivalence tests.
- [ ] Accept optional-evidence carrier and production/raw action split, with no missing-evidence error.
- [ ] Accept remove-on-missing replacement semantics, including temporary loss on an equivalent replacement whose capture fails.
- [ ] Accept PR17-origin binder refactoring in PR2 while retaining strict proof-side trust.
- [ ] Accept separate physical pruning follow-up and documented temporary memory growth.

**Implementation overview:** preserve ordinary validation, notify first, capture without additional I/O, log and omit on capture error, apply the payload unconditionally on evidence, and compare complete current Q only when processing proofs. This is deliberate minimal integration—not a claim of zero runtime overhead.
