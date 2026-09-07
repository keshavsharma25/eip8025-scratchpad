# Scratchpad — ProofEngine design (2026-09-07)

Base: Grandine `feature/sign-execution-proofs@9af0859`.
Spec: `feat/simplify-eip8025` tip `6946894b0`, base `7d6bd46a0`.
Design doc: `design/docs/design/0003-execution-proof-service.md`.
Archive: `~/ethereum/eth-rnd-archive`, `l1-zkevm-protocol`, pulled 2026-09-07.

## 1. What changed: stateful notify model → stateless 3-method model

Old spec (`321eca5`, copied by stale `0003 §1.1`):

```python
proof_state: ProofState
verify_execution_proof(proof) -> bool
notify_new_payload(request)                # "remember this payload"
notify_forkchoice_updated(head, safe, fin) # "head moved"
request_proofs(request, attrs) -> root
```

New spec (`proof-engine.md:26-40`):

```python
verify_execution_proof(execution_proof: ExecutionProof) -> bool
request_proofs(new_payload_request, chain_id, schema_id, attrs) -> Root
get_proof(new_payload_request_root, proof_type) -> ExecutionProof
# non-generating impls may reject request/get
```

No `ProofState`, no `notify_*`, no `stop` / `pow_block` / `exchange_capabilities`.
`request/get` did not replace `notify_*` — `notify_*` was deleted, and the
prover half (which always existed) gained explicit `chain_id`/`schema_id` args
plus a real `get_proof` poll instead of the old POST-callback.

`ExecutionProof = { proof_data, proof_type, public_input{
new_payload_request_root, successful_validation, chain_id, schema_id } }`.
CL builds it via service-owned `get_execution_proof`
(`store.payloads[block] + state.latest_execution_payload_bid +
DEPOSIT_CHAIN_ID + 0x1501`, `beacon-chain.md:202-234`) and hands the finished
object over.

## 2. Why notify_* died: filing stays with the CL

Human picture: old = assistant insists you forward every email so they can
file it, then digs through the cabinet when asked "is this valid?" New = you
hand them the complete folder and ask "check this"; they say yes/no and forget
it. Filing stays with you (the CL `Store`).

Concretely:

- **Two filing cabinets is one too many.** CL already tracks blocks, payloads,
  bids, `Seen`, head/safe/finalized. A second chain tracker inside the PE means
  two sources of truth — reorgs, backfill, optimistic import, range-sync
  bypassing the PE all become sync bugs. Feb-19 Lighthouse thread
  (`forkchoice_updated in LH impl`, `kevaundray`/`taulepton_`) reached the same
  conclusion: forking logic stays in CL; PE just gets told what to check. New
  model: no `ProofState`; `Store.execution_proofs` and `Seen` live in CL
  (`fork-choice.md`, `p2p-interface.md`).
- **No race.** Old model broke on ordering: gossip proof could arrive before
  the `notify`, or after a reorg. New model has none — if CL can't reconstruct
  from its own `Store`, it `IGNORE`s before ever calling the engine
  (`validate_execution_proof_gossip` order). Matches reality: zkboost verify is
  stateless HTTP, proof in body, bool out (`taulepton_` 2026-07-10); verify-only
  needs no `debug_executionWitness`.
- **notify bought nothing.** `notify_forkchoice_updated` exists to drive block
  building (`payload_attributes`) + optimistic sync. Proofs do neither
  (`beacon-chain.md:38-39` auxiliary, no head weight). The call was all cost —
  every block, every head update, even verifier-only — for zero function.

## 3. Prover vs verifier split

- **Verifier (everyone): just `verify`.** Pure crypto check over
  `hash_tree_root(public_input)`. One real method on a verifier-only node.
- **Prover (opt-in validators): async job.** `request_proofs(...) -> root`
  returns the job ID immediately; `get_proof(root, type)` waits and returns the
  finished proof. CL then wraps it in an envelope, signs it (`prover.md`), and
  gossips. Slow proving never blocks validation. Per Feb `Proof Generation`
  thread, this is subscription/poll on zkboost, not a POST callback, and avoids
  VC→BN requests.

## 4. What this means in Grandine: a structural twin, not a method twin

Yes, create the `proof_engine` crate — but `twin` means scaffolding twin.

Think plug-in, not twin engine. Grandine runs with the real thing, nothing, or
a fake for tests, without rewriting call sites:

- **Empty plug (`Null*`).** Same shape, does nothing. No EL? `NullExecutionEngine`
  (`execution_engine/src/execution_engine.rs:227`) answers `Ok`/`None`/ignore —
  used by bench controllers (`fork_choice_control/src/specialized.rs:52`) and
  back-sync. No verifier? Planned `NullProofEngine` makes the node ignore
  `execution_proof` gossip. Note the intentional asymmetry: EL Null is
  permissive, PE Null is fail-closed (`verify -> false`, prover stubs
  `Err(unsupported)`) but unreachable — the task short-circuits
  `IS_NULL -> Ignore` before any work, and the node subscribes to nothing.
- **Label on the plug (`IS_NULL`).** `const` flag surviving `&`/`Arc`/`Mutex`
  forwards, so generic code (`if E::IS_NULL`,
  `fork_choice_store/src/validations.rs:52`) skips PoW lookup / BLS / proof
  pipeline at compile time. No `Option<dyn>` unwrapping everywhere.
- **Fake plug with switches (`Mock*`).** `MockExecutionEngine::new(
  execution_valid, ...)` (`execution_engine.rs:317`): `true` passes, `false`
  rejects — happy + reject paths with no real EL. Planned `MockProofEngine`
  (`execution_proof_valid` + canned prover proof/error) does the same for the
  gossip task smoke test.

Crate layout (`lib.rs`, `engine.rs`, `null_engine.rs`, `mock_engine.rs`, no
`client.rs` until weeks 9–11) mirrors `execution_engine` because both are
external-system boundaries (`Arc<dyn _>` on the low-priority thread pool),
not because the methods match. Rejected: module in `fork_choice_control`
(loses boundary parity) and verify-only 1-method trait (forks from spec, breaks
when prover lands — 2 reject-stubs are cheaper). If `twin` overclaims, call it
`ExecutionEngine-patterned`.

## 5. Archive evidence (supports §§2–4, not repeated)

- **End goal, 2026-02-04 (`kevaundray`/`manunlp`):** same PE API as binary or
  future FFI/library (c-kzg style); docker now, library later. Justifies
  deferring `client.rs`.
- **process_execution_payload, 2026-02-04 (`taulepton_`, quoting #4828):**
  PE-only ⇒ EE no-op and vice versa. Separate-engine + no-op won over
  stateless-flag / merged-engine options.
- **Lighthouse forkchoice, 2026-02-19:** isolated-in-PE now, CL-owned fork logic
  long-term; proofs attest validity only (no invalidity proofs).
- **Prover flow, 2026-02-21..23:** poll/subscription model; PR #4943.
- **EL required?, 2026-04-15..23 (`barnabasbusa`/`taulepton_`/`jih2nn`):**
  conservative ship = EL required, LH no-EL research mode (optimistic import,
  k/n proofs gate attestation, EL wins conflicts). Tracked in #5140, #5151.
- **Nimbus/ethrex, 2026-03-18..30 (`deme1744`/`ivanlitteri`):** fork-independent
  conditional; proof-node API unspecced (`zkboost/.../client/src/lib.rs:49`,
  `api.json` gist, `proof-apis` vs `execution-apis` open); ethrex deleted ~3k
  lines of local PE once on zkboost.
- **Optional-proofs howto, 2026-07-10..13:** prover = EL + zkboost cluster;
  verifier-only = local zkboost sidecar, CL↔zkboost like CL↔EL, no public
  endpoint / no outsourced trust. Endgame: embed VKs in CL. No EL still
  research-only (`lighthouse:optional-proofs/testing/proof_engine`).
- **No `ethlambda` in archive** (only unrelated `post-quantum` hit) — assumed
  **ethrex**. No Discord debate on `notify_*` removal in last 2 months; that
  rationale lives in spec PRs (#5055, `67296e610`, #5566 announced 2026-09-01).

Artifacts: `eth-act/lighthouse:optional-proofs`, `eth-act/zkboost`,
`ere-guests`, consensus-specs #4828/#4943/#5014/#5055/#5140/#5151/#5566.
Open: no-EL spec, `proof-apis` home, mainnet proof gossip (none as of Jul-13).
