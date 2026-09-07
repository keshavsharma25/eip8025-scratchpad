# Execution proof pipeline — spec to Grandine mapping (2026-09-07)

Base: Grandine `feature/sign-execution-proofs@9af0859`.
Spec: `feat/simplify-eip8025` tip `6946894b0`.
Design doc: `design/docs/design/0003-execution-proof-service.md`.

## 1. The three spec functions

### 1.1 `validate_execution_proof_gossip` — `consensus-specs/specs/_features/eip8025/p2p-interface.md:80`

Signature: `(seen, store, signed_proof_envelope, proof_engine) -> None`, raises
`GossipIgnore` / `GossipReject`.

Ordered pipeline:

1. `[IGNORE]` proof not already processed (`seen.execution_proof_roots[block]`).
2. `[IGNORE]` beacon block seen (`store.blocks`).
3. `[REJECT]` beacon block consensus-valid (`store.block_states`).
4. `[IGNORE]` execution payload available (`store.payloads[block]`).
5. `[IGNORE]` no verified proof for `(block, proof_type)` (`store.execution_proofs`).
6. `[IGNORE]` prover first attempt for `(block, type, validator_index)`
   (`seen.execution_proof_provers`).
7. `[REJECT]` `verify_execution_proof_envelope(state, signed_envelope, payload_envelope)`.
8. Mark `seen.execution_proof_roots` + `seen.execution_proof_provers`.
9. `proof = get_execution_proof(state, envelope, payload_envelope)`.
10. `[REJECT]` `proof_engine.verify_execution_proof(proof)`.

This is the gossip entry point. It is the only one of the three that touches
`Seen` and the only one that encodes IGNORE vs REJECT.

### 1.2 `process_execution_proof` + helpers — `beacon-chain.md:171,199,236`

- `verify_execution_proof_envelope(state, signed_envelope, payload_envelope)`
  (`beacon-chain.md:171`): root equality, validator index bounds, proof size
  `0 < len <= MAX_PROOF_SIZE`, supported `proof_type`, active validator, BLS
  `DOMAIN_EXECUTION_PROOF` signature check.
- `get_execution_proof(state, proof_envelope, payload_envelope)`
  (`beacon-chain.md:199`): builds `SSZNewPayloadRequest` from
  `payload_envelope.payload + bid.blob_kzg_commitments (via
  kzg_commitment_to_versioned_hash) + parent_beacon_block_root +
  execution_requests`, hashes to `new_payload_request_root`, wraps with
  `proof_data/proof_type + successful_validation=True + DEPOSIT_CHAIN_ID +
  STATELESS_INPUT_SCHEMA_ID` into `ExecutionProof`.
- `process_execution_proof(state, signed_envelope, payload_envelope, proof_engine)`
  (`beacon-chain.md:236`): sequences the two above plus
  `proof_engine.verify_execution_proof(proof)`.

Pure on `(BeaconState, envelopes, ProofEngine)`. No `Store`, no `Seen`, no
IGNORE/REJECT. Called by both gossip validation and the fork-choice handler.

### 1.3 `on_execution_proof` — `consensus-specs/specs/_features/eip8025/fork-choice.md:100`

Signature: `(store, signed_envelope, proof_engine) -> None`.

1. Assert `beacon_block_root in store.blocks`, `state = store.block_states[block]`
   present, `payload_envelope = store.payloads[block]` present.
2. Assert no stored proof for `(block, proof_type)`.
3. Call `process_execution_proof(state, signed_envelope, payload_envelope,
   proof_engine)`.
4. Insert `store.execution_proofs[block][proof_type] = proof_envelope`.

No head change, no weight change, no state change. Proofs are stored
annotations.

## 2. Grandine placement

The "proof service" is split across `fork_choice_store` (validation logic +
storage) and `fork_choice_control` (async execution + serialized mutation),
mirroring bids/envelopes. The `ProofEngine` trait itself lives in the new
`proof_engine` crate and is injected as `Arc<dyn ProofEngine>`.

### 2.1 Validation logic — `grandine/fork_choice_store/src/store.rs`

New `Store::validate_execution_proof(&self, envelope, proof_engine)` sits next
to `validate_execution_payload_bid:2222` and
`validate_execution_payload_envelope:3777`. It implements §1.1 verbatim against
the snapshot: state from `store.block_states`, payload from `store.payloads`,
dedup from `store.execution_proofs` plus the new seen sets (Grandine folds
spec `Seen` into store-side `accepted_*` maps, e.g. `accepted_payload_bids:273`).
Returns `Result<ExecutionProofAction>` with `Ok(Ignore)` vs `Err(Reject)`
preserving spec IGNORE/REJECT. The `proof_engine.verify` call stays inside
because the caller runs it off-thread.

The §1.2 helpers live beside it as pure functions on `(&BeaconState,
envelopes, &impl ProofEngine)`, called from both validation and `on_*`.

### 2.2 Async task — `grandine/fork_choice_control/src/tasks.rs`

New `ProcessExecutionProofTask` mirrors `ExecutionPayloadBidTask:701`:

```rust
pub struct ProcessExecutionProofTask<P, E> {
    pub store_snapshot: Arc<Store<P, Storage<P>>>,
    pub mutator_tx: Sender<MutatorMessage<P, W>>,
    pub envelope: Arc<SignedExecutionProofEnvelope>,
    pub origin: ExecutionProofOrigin,
    pub proof_engine: Arc<E>,
}
```

`run()` calls `store_snapshot.validate_execution_proof(...)` — i.e. §1.1, which
internally calls §1.2 — and sends `MutatorMessage::ExecutionProof { result,
origin }`. Contrast `ExecutionPayloadEnvelopeTask:649`, which calls
`validate_execution_payload_envelope` and forwards the richer envelope message.

### 2.3 Message — `grandine/fork_choice_control/src/messages.rs`

New variant follows `MutatorMessage::PayloadBid:172-175` (`{ result, origin }`,
no `wait_group`), not `ExecutionPayloadEnvelope:159` or `Block:88` or
`PayloadAttestation:176`, all of which carry `wait_group: W`. Bids and proofs
gate nothing downstream (no block import, no API response waits), so the task
is fire-and-forget and the mutator needs no join handle.

### 2.4 Mutation — `grandine/fork_choice_control/src/mutator.rs`

New `handle_execution_proof(result, origin)` beside `handle_payload_bid:359`
implements §1.3: on `Ok(Accept)` re-check single-proof-per-type, insert into
`store.execution_proofs`, refresh the snapshot; on `Ignore`/`Reject` drop and
report for peer scoring.

## 3. Data flow

Gossip `SignedExecutionProofEnvelope` → controller spawns
`ProcessExecutionProofTask` with snapshot + engine → `validate_execution_proof`
(full §1.1 order, §1.2 tail, engine verify off-thread) →
`MutatorMessage::ExecutionProof` → mutator `handle_execution_proof` (§1.3
insert). Spec order is preserved word-for-word in the store validation
function; the task and mutator are thin wrappers for threading and
serialization.
