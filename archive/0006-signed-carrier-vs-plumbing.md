# Scratchpad — Signed carrier vs plumbing, Grandine way (2026-09-07)

Base: Grandine `feature/sign-execution-proofs@9af0859`.
Spec: `feat/simplify-eip8025` tip `6946894b0`, base `7d6bd46a0`.
Design doc: `design/docs/design/0003-execution-proof-service.md`.
Sequel to `0004-proof-engine-discussions.md` (§2 filing stays with CL)
and `0005-execution-proof-pipeline.md` (§1.3 `on_execution_proof`, §2 placement).

This note answers one confused sentence in 0003 Goals:

> Spec-fidelity for the stored shape: `Store.execution_proofs` mirrors
> `fork-choice.md` exactly; surrounding plumbing follows Grandine convention.

The confusion is natural because three different things all look like
"the proof": the thing we store, the thing we pass around, and the pipes
it flows through. They are not the same.

## 1. Stored shape vs carrier vs plumbing

Human picture: think of a courthouse.

- The **stored shape** is the file in the archive box. The spec dictates
  the folder label and paper size. Every courthouse uses the identical box
  or files stop being interchangeable.
- The **carrier** is the envelope the courier carries through the building.
  It holds the file plus a cover sheet (who delivered it, whose signature
  is on the receipt). The cover sheet never goes into the archive box.
- The **plumbing** is the building itself: reception desk, intake queue,
  clerk's desk, intercom back to reception. Every courthouse lays its
  corridors out differently.

Concretely:

| Layer | Type | Decided by |
|---|---|---|
| Stored shape | `ExecutionProofEnvelope` (bare) | spec, verbatim |
| Carrier | `SignedExecutionProofEnvelope` (bare + `validator_index` + `signature`) | Grandine plumbing choice |
| Plumbing | task + message + origin + mutator arm + thread pool slot | Grandine convention |

Mixing these up produces two classic bugs: inventing a custom stored
shape ("we will store the signed envelope, it is more convenient"), or
copy-pasting spec Python structure into Rust where it cannot run
("we will call `on_execution_proof` directly on the live store from
the gossip thread").

## 2. Stored shape: the one place we do not get a vote

`consensus-specs/specs/_features/eip8025/fork-choice.md:32-53`:

```python
# [New in EIP8025]
execution_proofs: Dict[Root, Dict[ProofType, ExecutionProofEnvelope]]
```

Rust mirror (`0003:199-205`):

```rust
// [New in EIP8025]
execution_proofs: HashMap<H256, HashMap<ProofType, ExecutionProofEnvelope>>,
```

- Outer key is `beacon_block_root`. Inner key is `proof_type`. Value is
  the **bare** envelope. Not the signed envelope, not the engine-facing
  `ExecutionProof` (which also carries `public_input`).
- Initialized empty (`fork-choice.md:86` → `execution_proofs={}` in
  `get_forkchoice_store`).
- Inserted only after `process_execution_proof` passes
  (`fork-choice.md:123-128` — "Store only proofs that pass downstream
  verification"). One verified proof per `(block, type)`.
- Stored annotations only (`fork-choice.md:22-23,94-97`): no head change,
  no weight change, no state change, no Gloas payload status change.

Why strict: storage shape is consensus-visible. Sync (`ByRange`),
k-of-n counting, and cross-client test vectors all assume the same
drawer dimensions. So no Grandine creativity here.

## 3. Signed carrier: what actually travels task → Accept → mutator

Gossip gives us `SignedExecutionProofEnvelope`:

```rust
SignedExecutionProofEnvelope {
    message: Hc<ExecutionProofEnvelope>, // bare envelope, hashed + cached
    validator_index: ValidatorIndex,     // who is attesting to this proof
    signature: SignatureBytes,           // DOMAIN_EXECUTION_PROOF over message
}
```

The design (`0003:166-169,192-197`, trade-offs `0003:288`) carries this
**signed** object end to end — task → `Accept` → mutator — and strips to
bare `.message` only at the store boundary:

```
task holds Arc<SignedExecutionProofEnvelope>
  → validate (envelope auth + engine verify)
  → MutatorMessage::ExecutionProof { result: Ok(Accept(signed)), origin }
  → mutator: store.execution_proofs[block][type] = signed.message.0
```

Why not carry the bare envelope from the start:

- **`Seen` marking needs `validator_index`.** Spec step 8–9 in
  `0005 §1.1`: mark `seen.execution_proof_provers[(root, type,
  validator_index)]` after envelope auth, before engine verify. The bare
  envelope has no `validator_index`. Drop it early and you cannot mark.
- **Logging, events, and peer scoring need the signer.** Reject/ignore
  paths feed peer scoring via `origin`; accept-path events want to say
  *whose* proof arrived, not just *what* proof.
- **The store stays spec-exact either way.** Both accept variants end with
  the identical bare envelope in the map. Carrying the signed object one
  hop longer costs one `Arc` clone and buys back the cover sheet.

Human picture: the courier keeps the cover sheet until the clerk files
the paper. The archive box never contains cover sheets.

What `Accept` does **not** mean: "we minted a fresh proof ready to gossip
to other peers." This workstream is verifier-only (`0003:60-67`
non-goals). `Accept` means "the gossip we *received* passed, file the bare
copy and tell p2p to keep propagating the signed copy." Prover-originated
gossip (`request_proofs` / `get_proof` flow) does not exist yet.

## 4. Plumbing, Grandine way: same checks, different pipes

The spec is single-threaded Python: `on_execution_proof(store, signed,
engine)` asserts and inserts as one atomic step (`fork-choice.md:100-129`).
Grandine cannot do that — gossip arrives on network threads while the head
is moving. So Grandine splits every gossip intake into the standard
low-priority invariant:

> snapshot-validate concurrently → `Result<Action>` + `Origin` →
> mutator applies serially to `Store` + signals p2p.

For proofs the walk is (`0005 §2`, `0003:71-97`):

```
eth2_libp2p (gossip) → p2p router → Controller
                                  │ spawn_execution_proof_task()
                                  ▼
                  ProcessExecutionProofTask::run()   # LowPriorityTask
                    store_snapshot.validate_execution_proof(...)
                      = p2p-interface.md:80-151 verbatim, in spec order:
                        seen-root IGNORE → block-seen IGNORE
                        → block-valid REJECT → payload-available IGNORE
                        → type-new IGNORE → prover-new IGNORE
                        → verify_execution_proof_envelope REJECT
                        → service-owned get_execution_proof (build ExecutionProof)
                        → mark Seen → engine verify_execution_proof REJECT
                                  │
                                  ▼
            MutatorMessage::ExecutionProof { result, origin }
                                  │
                     mutator arm → apply to Store + signal p2p
```

Each pipe follows an existing precedent, deliberately:

- **Validation logic** — `Store::validate_execution_proof(&self, ...)`
  in `fork_choice_store/src/store.rs`, next to
  `validate_execution_payload_bid:2222` and
  `validate_execution_payload_envelope:3777`. Reads state from
  `store.block_states`, payload from `store.payloads`, dedup from
  `store.execution_proofs` plus the new seen sets (Grandine folds spec
  `Seen` into store-side `accepted_*`-style maps, e.g.
  `accepted_payload_bids:273`). Returns `Result<ExecutionProofAction>`
  with `Ok(Ignore)` vs `Err(Reject)` preserving spec IGNORE/REJECT. The
  slow `proof_engine.verify` call stays inside because the caller runs it
  off-thread.
- **Task struct** — `ProcessExecutionProofTask` in
  `fork_choice_control/src/tasks.rs`, cloned from
  `ExecutionPayloadBidTask:701-722` (`store_snapshot + mutator_tx +
  payload_bid + origin` → `validate_*` → `MutatorMessage::PayloadBid`
  send). Same four fields, envelope swapped in, engine `Arc` added.
- **Thread-pool slot** — `LowPriorityTask::ExecutionProof` variant in
  `fork_choice_control/src/thread_pool.rs:152-169`, beside
  `PayloadBid(...)`. Proofs are up to 4 MiB plus BLS plus a potentially
  slow external verify, so they never run on the critical path.
- **Message** — bid-style `MutatorMessage::ExecutionProof { result,
  origin }`, following `PayloadBid:172-174` (`{ result, origin }`, no
  `wait_group`), not `ExecutionPayloadEnvelope:159-162` or `Block:88-95`
  or `PayloadAttestation:176`, all of which carry `wait_group: W`. Bids
  and proofs gate nothing downstream (no block import, no API response
  waits), so the task is fire-and-forget and the mutator needs no join
  handle.
- **Origin** — new `ExecutionProofOrigin(GossipId)`, mirroring
  `ExecutionPayloadBidOrigin`, not a raw `GossipId`, so the mutator can
  `origin.split()` for p2p accept/reject/ignore signalling.
- **Mutation** — `handle_execution_proof(result, origin)` in
  `fork_choice_control/src/mutator.rs`, beside `handle_payload_bid:359 /
  2514`. On `Ok(Accept)` it re-checks single-proof-per-type (a second
  proof may have landed between snapshot and apply), inserts the bare
  `.message` into `store.execution_proofs`, refreshes the snapshot. On
  `Ignore`/`Reject` it drops and reports for peer scoring.
- **Null short-circuit** — `run()` opens with `if E::IS_NULL → Ignore`
  before any pipeline work (`0004 §4` twin pattern: `NullExecutionEngine`
  at `execution_engine/src/execution_engine.rs:227`, `IS_NULL` flag
  surviving `&`/`Arc`/`Mutex` forwards). Opt-out nodes subscribe to
  nothing and never pay for verification.

`get_execution_proof` placement is part of the same plumbing call: it is
service-owned (pure function of `(state, envelope, payload_envelope)` —
payload + `bid.blob_kzg_commitments` + `DEPOSIT_CHAIN_ID` +
`STATELESS_INPUT_SCHEMA_ID`), not engine-internal, because all its inputs
are consensus-layer state (`beacon-chain.md:199-234`).

## 5. One-line decoder ring

> Store the bare envelope exactly like the spec says. Carry the signed
> envelope through Grandine-shaped pipes to get it there.
