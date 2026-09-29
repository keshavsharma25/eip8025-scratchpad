# Scratchpad — 0003 Execution-Proof Service (review-first, verbose)

Status: grill Q1–Q11 agreed 2026-09-06. §4 lock-ins LOCKED 2026-09-07 (see §4). Design doc update comes AFTER this scratchpad review. Implementation starts only after doc agreed.
Base: Grandine `feature/sign-execution-proofs@9af0859`. 0003 work branch stacks directly on it (no rebase onto develop).
Design doc: `/home/keshav-wsl/code/8025/design/docs/design/0003-execution-proof-service.md` (stale, pins `321eca5` / `eaf220e`).
Spec: local `/home/keshav-wsl/code/8025/consensus-specs @ feat/simplify-eip8025 tip 6946894b0`; doc base citation `7d6bd46a015a7dd316c5df855bd89e57c4aa6700`.

## 1. Starting state (what is stale and what is current)

### 1.1 Design doc 0003 as written

- Pinned to old spec `321eca5`, old EIP text, Grandine `eaf220e`.
- Assumes generic `ExecutionProof<P>` / `SignedExecutionProof<P>` in `types/src/eip_8025/containers.rs`.
- `ProofEngine` as `ExecutionEngine` twin: `verify(ExecutionProof)->bool` + `notify_new_payload(NewPayloadRequest)` + `notify_forkchoice_updated(head,safe,finalized)` + `stop`, with `Null/Mock`, context bound engine-internally from notify history.
- Dedup: service-owned bounded LRU keyed `(new_payload_request_root, proof_type, validator_index)`, marked before validity.
- Pipeline: bounds → dedup → eligibility → BLS → context-resolve → engine.
- Return: `MutatorMessage::ExecutionProof { result: ProofOutcome }` outcome-only (`Accept/Reject/Ignore`), identifiers deferred to weeks 15+.
- zkboost wire body `(fork, new_payload_request_root, chain_config, proof_type, proof bytes)` assembled engine-side.
- Blockers say containers don't exist; open question says `MAX_PROOF_SIZE` enters `Preset` as `P::MaxProofSize`.

### 1.2 Current Grandine (`feature/sign-execution-proofs@9af0859`)

- `types/src/eip8025/containers.rs`: non-generic `ExecutionProof { proof_data, proof_type, public_input }` (`containers.rs:130-137`); 4-field progressive `PublicInput { new_payload_request_root, successful_validation, chain_id: u64, schema_id: u16 }` (`containers.rs:114-124`, `#[ssz(stable(active=[1;4]))]`); gossiped `ExecutionProofEnvelope { proof_data, proof_type, beacon_block_root }` (`containers.rs:146-153`); `SignedExecutionProofEnvelope { message: Hc<ExecutionProofEnvelope>, validator_index, signature }` (`containers.rs:162-169`); `SszNewPayloadRequest<P>` progressive (`containers.rs:187-198`).
- `types/src/eip8025/consts.rs`: `MAX_PROOF_SIZE` via `MaxProofSize::USIZE` (`consts.rs:24`); `STATELESS_INPUT_SCHEMA_ID = 0x1501` (`consts.rs:34`); `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE = 108 + 37 + MAX_PROOF_SIZE` (`consts.rs:47`); `DOMAIN_EXECUTION_PROOF = 0x0F000000` (`consts.rs:50`).
- `types/src/eip8025/primitives.rs`: `ProofType = u8`; `MaxProofSize = U4194304` (`primitives.rs:1-8`).
- `ProofData` wraps `ProgressiveByteList<MaxProofSize>` (`containers.rs:34-38`); root ignores bound; bound enforced in `TryFrom<Vec<u8>>`/decode/serde (`containers.rs:40-107`); no `From<ProgressiveByteList>` bypass.
- Signing retargeted to envelope: `helper_functions/src/signing.rs:530-533` `impl SignForSingleForkAtSlot for ExecutionProofEnvelope` (`DOMAIN_EXECUTION_PROOF`, `SignatureKind::ExecutionProof`); doc link pinned to `7d6bd46` prover `get_execution_proof_envelope_signature`.
- No `proof_engine` crate yet. No `ProcessExecutionProofTask` yet.

### 1.3 Simplify-branch spec (source of truth)

- `beacon-chain.md`: `ProofData = ProgressiveList[Byte]`; `ProofType = Uint8`; `MAX_PROOF_SIZE = 4194304`; `STATELESS_INPUT_SCHEMA_ID = 0x1501`; `DOMAIN_EXECUTION_PROOF = 0x0F000000`; `SSZNewPayloadRequest` / `PublicInput` progressive width-4; `ExecutionProof` (engine-facing) / `ExecutionProofEnvelope` (gossiped) / `SignedExecutionProofEnvelope`; `get_supported_proof_types() = {1,2,3}` provisional; `verify_execution_proof_envelope` (root match, index bounds, `0<len≤MAX`, type allowlist, active validator, BLS); `get_execution_proof` (build `PublicInput` from payload + `DEPOSIT_CHAIN_ID` + schema); `process_execution_proof` (auth then `verify_execution_proof`).
- `proof-engine.md`: `ProofEngine` = `verify_execution_proof(ExecutionProof)->bool` + `request_proofs(new_payload_request, chain_id, schema_id, attrs)->Root` + `get_proof(root, type)->ExecutionProof`. Generation/retrieval prover-only; non-generating impls "may reject".
- `prover.md`: `get_execution_proof_envelope_signature` (domain at `compute_epoch_at_slot(state.slot)`); `request_execution_proofs`; `get_signed_execution_proof_envelope` (assert public-input fields, wrap envelope, sign, broadcast on `execution_proof` topic).
- `p2p-interface.md`: gossip-only, no Req/Resp; `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE = 4194449`; `Seen { execution_proof_roots: Dict[Root, Set[Root]], execution_proof_provers: Set[(Root, ProofType, ValidatorIndex)] }`; `validate_execution_proof_gossip` order: seen-root IGNORE → block-seen IGNORE → block-valid REJECT → payload-available IGNORE → type-new IGNORE → prover-new IGNORE → envelope-auth REJECT → build proof → mark seen → engine REJECT.
- `fork-choice.md`: `Store.execution_proofs: Dict[Root, Dict[ProofType, ExecutionProofEnvelope]]`; `on_execution_proof` asserts block known + valid + payload available + type-new, calls `process_execution_proof`, stores bare envelope. No weight/head/state/Gloas change.

### 1.4 Diff commands (for reviewer)

```bash
cd /home/keshav-wsl/code/8025/consensus-specs && git fetch origin
git log --oneline --follow -- specs/_features/eip8025/proof-engine.md
git diff 321eca5b71049fcac6c63c2d956e5c5d7b60d689..6946894b0 -- specs/_features/eip8025/proof-engine.md
git diff 321eca5b71049fcac6c63c2d956e5c5d7b60d689..6946894b0 -- specs/_features/eip8025/
git show 321eca5b71049fcac6c63c2d956e5c5d7b60d689:specs/_features/eip8025/proof-engine.md
git show 6946894b0:specs/_features/eip8025/proof-engine.md
```

Notable history: `768268ae1 Refactor proof engine and p2p interface (#5055)`, `67296e610 simplify proof processing`, `3252266dd align public inputs with execution specs`, `c15e877f3 Skip items` / revert `2d9fda8b7`.

### 1.5 Discord `l1-zkevm-protocol` (archive `~/ethereum/eth-rnd-archive`, pulled 2026-09-06)

- `proof engine(verifier): end goal` (2026-02-04 kevaundray/manunlp): verifier starts as same-API binary/docker, FFI/library later like c-kzg → justifies deferring `client.rs`.
- `Default to execution engine and proof engine` (2026-04-15..23 barnabasbusa/taulepton_/jih2nn/kevaundray): EL stays required for optional proofs (conservative; issue #5140 tracks no-EL future); Lighthouse no-EL mode gates attestation on k/n proofs → keep `Null` short-circuit story, spec still asserts EL.
- `Proof Generation & Singing Flow` (2026-02-21..23): `request_proofs` should be subscription/poll on zkboost, not POST callback → shapes prover-method semantics.
- `Validator Proof Resigning` (2026-04-20..22 taulepton_/jih2nn/jtraglia): resigning + blacklist dropped; p2p scoring + devnet testing instead → doc must not add resigning.
- `Consideration on StatelessInput` (2026-07-08..10 jsign/deme1744/0xrubo): `blob_schedule` + `fork` removed from chain config; fork moves into `SCHEMA_ID` high byte (`0x1501` = Amsterdam/01) → why `PublicInput { chain_id, schema_id }` exists and CL builds it locally.
- `Consensus specs PR and discussion` (2026-03-17..24): fork specs removed (#5037); `ExecutionProofStatus` replaces Metadata; `ExecutionProofsByRange` mirrors blobs; gossip-only.
- `Proof type encoding` (2026-05-04..13): `u8 {1,2,3}` provisional; human-readable `ethrex-<commit>/zisk-vX` mapping in `eth-act/execution-proofs-api#1`.
- Wikipethia MCP tools unavailable in this runtime (only `read/grep/shell/webfetch/websearch`); no corpus query run. Use websearch/webfetch or an MCP-enabled agent if needed.

## 2. Agreed decisions Q1–Q11 (binding for doc + implementation)

- Q1 verifier-only: `request_proofs`/`get_proof` are prover-role; Grandine 0003 wires verification only.
- Q2 full trait with reject: `ProofEngine` keeps all 3 spec methods; prover methods default-reject per `proof-engine.md:26-40`. No shrink-to-1 (avoids breaking change later).
- Q3 service-owned `get_execution_proof`: task reconstructs `SszNewPayloadRequest` → `PublicInput` → `ExecutionProof` from `store.payloads[beacon_block_root]` + `state.latest_execution_payload_bid` + `DEPOSIT_CHAIN_ID` + `STATELESS_INPUT_SCHEMA_ID`; engine takes full proof; notify-history and zkboost wire-body stories deleted.
- Q4 spec `Seen` + store: `execution_proof_roots` + `execution_proof_provers` keyed by `beacon_block_root` (not `new_payload_request_root`) + `store.execution_proofs[block][type]`; mark after envelope auth, before engine verify; LRU bounding is impl detail.
- Q5 spec-ordered pipeline in `run()`: seen-root IGNORE → block-seen IGNORE → block-valid REJECT → payload-available IGNORE → type-new IGNORE → prover-new IGNORE → `verify_execution_proof_envelope` REJECT → build → mark → engine REJECT.
- Q6 Null/Mock: `Null { IS_NULL=true, verify=false, request/get → Err(unsupported) }` + task `IS_NULL → Ignore` short-circuit so Null never Rejects; `Mock { execution_proof_valid: bool }` with canned prover proof/error for smoke tests.
- Q7 Bid-style return + invariant line (MUST appear in doc): low-priority gossip invariant is "task validates on snapshot → returns `Result<Action>` + `Origin(GossipId)` → mutator applies to `Store` and signals p2p." Hence `MutatorMessage::ExecutionProof { result: Result<ExecutionProofAction>, origin }`, `Accept` carries signed envelope (mutator stores bare `.message` per spec), `Ignore(reason)`, `Err = Reject`. Evidence: `AggregateAndProofTask` (`tasks.rs:291-325` → `mutator.rs:1161-1230` with `apply_attestation` + `P2pMessage::Accept/Ignore`), `AttestationTask` (`tasks.rs:328-357`), `AttesterSlashingTask` (`tasks.rs:464-497` → `mutator.rs:1678-1708`), `ProposerPreferencesTask` (`tasks.rs:804-831`); `PayloadBid` (`tasks.rs:701-722` → `mutator.rs:2514-2575`) is the Gloas instance minus `wait_group`. Envelope-style (`+wait_group/timings/tracing/persistence`, `thread_pool.rs:117`, `messages.rs:159-167`) rejected: wrong priority class.
- Q8 naming+pins on `sign-execution-proofs@9af0859` base: non-generic envelope set, `types::eip8025` path, `ExecutionProofAction` (not bespoke `ProofOutcome`), `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE`; front-matter → spec `7d6bd46` / tip `6946894b0`, Grandine `feature/sign-execution-proofs@9af0859`.
- Q9 bounds: 3 layers — `ProofData` decode bound (`containers.rs:40-107`, `primitives.rs:8`) + envelope `0<len≤MAX` gate + gossip pre-decode cap (`consts.rs:47`); root independent of bound (`containers.rs:20-38`); provisional `{1,2,3}` in envelope gate; `Preset::MaxProofSize` idea dropped.
- Q10 non-goals: subscription wiring, `ExecutionProofStatus`/`ByRange`, k-of-n counting/threshold, recursive anchors, retention/pruning/restart re-derivation, `client.rs` (weeks 9–11), prover flow. Skeleton stores one envelope per `(block,type)` with no eviction.
- Q11 work order (stacked on `sign-execution-proofs`): (1) `proof_engine` crate scaffold + workspace member, gate `cargo check -p proof_engine --features bls/blst`; (2) `fork_choice_control` stub (`ExecutionProofAction`, message variant, `ProcessExecutionProofTask` stub, `spawn_execution_proof_task` mirroring `controller.rs:987-998`, `LowPriorityTask::ExecutionProof` in `thread_pool.rs:152-169` + `Spawn` impl like `thread_pool.rs:269-273`, mutator arm); (3) Null/Mock unit + task smoke tests. Gates per 0002 §7 (`--features bls/blst`, fmt, clippy allows).

## 3. Doc rewrite checklist (section by section)

- Front-matter: status/authors/workstream keep; `eip_sha`/`consensus_specs_sha` → `7d6bd46` (+ simplify tip note); `grandine_upstream_sha` → `9af0859` + `base_branch: feature/sign-execution-proofs` + stacked-branch list.
- Context: delete zkboost wire-body contract + notify-history context resolution; replace with envelope/gossip/store summary + `Seen` + `on_execution_proof`; fix `types/src/eip_8025` → `types/src/eip8025`; fix container list to 5 non-generic types; cite `prover.md`/`beacon-chain.md`/`p2p-interface.md`/`fork-choice.md` + Discord threads above.
- Goals: `proof_engine` crate with 3-method trait + forwards + Null/Mock (prover reject); `ProcessExecutionProofTask` Bid-style + `ExecutionProofAction`; lock boundary (service does Seen/store/envelope-auth/public-input build, engine does crypto verify).
- Non-goals: Q10 list verbatim.
- Design: engine trait code block (verify + request/get reject defaults); `get_execution_proof` ownership; pipeline numbered list in Q5 order with IGNORE/REJECT labels; message/action code blocks (Bid-style) + invariant line + carrier rationale (signed carries bare); crate layout (`engine.rs`/`null_engine.rs`/`mock_engine.rs`, `client.rs` deferred); task struct (`store_snapshot, proof_engine: Arc<E>, mutator_tx, signed_proof: Arc<SignedExecutionProofEnvelope>, origin`) + `spawn_execution_proof_task` + `LowPriorityTask` variant; ownership summary (service: Seen/store/checks/build; engine: crypto verify + future retained state).
- Trade-offs: crate vs module (parity with `execution_engine`, `execution_engine/src/execution_engine.rs:22`); full-trait vs verifier-only (spec-faithful, reject defaults); Seen+store vs LRU (spec keys, LRU only as bound); Bid-style vs Envelope-style (priority class + gossip invariant); Action vs ProofOutcome (Grandine convention).
- Security: cheapest-first + pre-validity dedup; BLS+eligibility before engine; failed proof never invalidates Engine-API payload; Null opt-out subscribes to nothing; no resigning/blacklist per Discord decision.
- Implementation/testing: Q11 3 steps + gates; stub-now/replace-later note (checks weeks 9–11, gossip 12–14, p2p routing 15–16, anchors 17–19); zkboost mock backend note stays but behind trait.
- Blockers: containers + signing resolved (cite `9af0859`); `Store.execution_proofs` field + `apply` stub remain as implementation work; `Seen` location and `Origin` type LOCKED per §4 (2026-09-07).
- Open questions: kill `Preset::MaxProofSize` + signing-ownership + envelope-shape items; keep: Seen bound const value, k-of-n threshold values, recursive layer re-pin, no-EL future (#5140), proof-type registry (`execution-proofs-api#1`).

## 4. §4 lock-ins — LOCKED 2026-09-07 (binding for doc + implementation)

1. Carrier — LOCKED: `Accept(Arc<SignedExecutionProofEnvelope>)`. Task receives signed
   from gossip; mutator stores `signed.message` (bare) per spec `on_execution_proof`
   (unwrap at top, store bare at tail). Keeps `validator_index`/signature for
   `Seen.provers` marking, logging/metrics, event fan-out (cf. `handle_payload_bid`,
   `mutator.rs:2521-2526`), future k-of-n. Cost: one `Arc` clone + ~104 B vs ≤4 MiB
   proof; `Hc::clone` carries the cached root. Rejected: bare-carrier (simpler type,
   but forces re-threading `validator_index` the day attribution is needed).
2. `Seen` — LOCKED: Store-owned bounded mirrors of `execution_proof_roots` +
   `execution_proof_provers`, beside `seen_gossip_*` (`store.rs:172-176`); checked in
   `validate_*`, marked in `apply_*`. Bounding is impl detail. Rejected: p2p-layer cache
   (splits dedup from Store, double-verify races across snapshots).
3. `wait_group` — LOCKED: omit (Bid-style). `MutatorMessage::ExecutionProof { result, origin }`,
   no latch — proofs are fire-and-forget gossip like bids. Re-add is one line if a test
   ever needs a barrier.
4. `Store.execution_proofs` type — LOCKED: `HashMap<H256, HashMap<ProofType, ExecutionProofEnvelope>>`
   (verbatim spec `Dict[Root, Dict[ProofType, Envelope]]`; `ProofType` alias, not bare `u8`,
   per `primitives.rs:2`). Bare values, one entry per `(block, type)`, plain insert in
   `apply`, no eviction in skeleton; pruning rides on block finalization later.
5. `Origin` — LOCKED: new `ExecutionProofOrigin` on the `ExecutionPayloadBidOrigin` template
   (`misc.rs:479-519`): `Gossip(GossipId)` + `Api(OneshotSender<...>)` with `split()` /
   `gossip_id()`. Verified not to exist yet (grep 2026-09-07). Skeleton constructs only
   `Gossip` (`controller.rs:413` pattern).

## 5. Key file:line anchors

- Grandine containers/consts/primitives: `types/src/eip8025/containers.rs:34-38,40-107,114-124,130-169,187-198`; `types/src/eip8025/consts.rs:24,34,47,50`; `types/src/eip8025/primitives.rs:1-8`.
- Signing: `helper_functions/src/signing.rs:28,530-533`; error `SignatureKind::ExecutionProof`.
- Engine twin: `execution_engine/src/execution_engine.rs:22-56` (trait), `58-224` (forwards), `226-265` (Null), `267-335` (Mock).
- Tasks/messages/pool/mutator/controller: `fork_choice_control/src/tasks.rs:291-357,464-497,701-722,804-831`; `messages.rs:79-228,244-257`; `thread_pool.rs:108-191,269-273`; `mutator.rs:234-280,342-361,1161-1230,1678-1708,2132-,2514-2575`; `controller.rs:987-998,1024-1051`.
- Spec: `specs/_features/eip8025/{beacon-chain.md,proof-engine.md,prover.md,p2p-interface.md,fork-choice.md}` @ `6946894b0`.
- Plans: `/home/keshav-wsl/.opencode/plan/0002-sign-execution-proofs-plan.md` §7 gates, §10 re-target.

## 6. Learnings appendix

Verbose cross-conversation memory for the 9 selected topics. Binding decisions stay in §4; this section is rationale.

### 6.1 Store purpose — working memory of the chain

`Store` answers "what is the head right now, and what do I need to decide the next head?" Persistent DB (`Storage`) archives history; `Store` holds the live fork-choice view:

- Block DAG: `finalized: Vector<ChainLink<P>>` + `unfinalized: OrdMap<SegmentId, Segment<P>>` (`fork_choice_store/src/store.rs:138-146`).
- Justification/finalization: `justified/finalized/unrealized` checkpoints (`store.rs:123-127`).
- Votes: `latest_messages: Vector<Option<Arc<LatestMessage>>>` (`store.rs:166`). Head = heaviest LMD-GHOST + proposer boost (`proposer_boost_root`, `store.rs:135`).
- Availability facts: `payloads / timely_payloads: HashSet<H256>` (`store.rs:196-197`), `payload_vote*` maps, blob/column/bid/envelope acceptance maps.
- Query caches: `checkpoint_states`, `state_cache`, gossip-seen sets.

What it is not: not the DB, not the state-transition function (`BlockProcessor`), not the EL, not the signer. It records outcomes so fork choice stays a pure function of small in-memory state. For proofs: `execution_proofs[block][type]` is one more availability fact — no head weight.

### 6.2 Store internals — fields, snapshot, validate/apply, Store vs Storage

Two halves in one struct (`store.rs:118-295`): consensus-critical (head inputs) and availability/dedup caches (no head effect). Proofs belong to the second group.

Snapshot model (`fork_choice_control/src/controller.rs:1-9`, `:89`, `:160`): `Controller` holds `store_snapshot: Arc<ArcSwap<Store>>`. Every `on_*` clones it (`owned_store_snapshot()`, ~20 call sites) into a task. Tasks validate with `&self` reads only — lock-free, parallel. Mutator (single thread owning `Store` mutably) applies verdicts serially via `store_mut().apply_*()` then `update_store_snapshot()` publishes the new `Arc<Store>`. In-flight tasks finish on the old snapshot; redundant processing is expected (`controller.rs:8-9`).

Validate/apply split (`store.rs:1828+` vs `store.rs:4249+`): `validate_execution_payload_bid(&self, ...) -> Result<Action>` (`store.rs:2222`) does slot window → dedup (`accepted_payload_bids`, `store.rs:2263-2285`) → state lookup → BLS → `Accept(bid)`. `apply_execution_payload_bid(&mut self, ...)` (`store.rs:4725-4738`) is a plain insert. Same for `apply_execution_payload_envelope` (`store.rs:4686-4705`). Proofs follow identically: `validate_execution_proof` (spec gossip order) + `apply_execution_proof` (bare insert + `Seen` mark).

Store vs Storage: `storage: Arc<S>` inside `Store` (`store.rs:283`) is the persistent handle; `Controller`/`Mutator` flush on stop (`controller.rs:104-109`). Heavy replayable data lives in `*_cache` structs with own eviction; `Store` keeps roots/indices/commitments. Proofs (≤1 per `(block, type)`, ≤4 MiB each) belong in `Store` proper. Pruning rides on finalization; no independent LRU in skeleton.

### 6.3 Task→mutator channel — readers hand verdicts to one writer

Why: `&mut Store` cannot be shared across threads. Many reader threads validate against frozen snapshots; one writer applies serially. The channel is the handoff.

Actors: snapshot `Arc<ArcSwap<Store>>` (`Arc` = shared pointer, clone bumps counter; `ArcSwap` = atomic publish box). Task (`tasks.rs`, e.g. `ExecutionPayloadBidTask:701-722`): `{ store_snapshot, mutator_tx, object: Arc<Signed...>, origin }`, one-shot `Run::run(self)`. Channel `mutator_tx: Sender<MutatorMessage>` — multi-producer, single-consumer. Mutator (`mutator.rs`): `Accept` → `apply_*` + publish + p2p reply; `Ignore/Reject` → p2p reply only. Serialization is the synchronization — only this loop calls `store_mut()`.

Journey of one proof: (1) receive: p2p decodes `SignedExecutionProofEnvelope`, wraps in `Arc`, builds `origin`; (2) spawn: `spawn_execution_proof_task` clones snapshot + `mutator_tx`, pushes `ProcessExecutionProofTask` onto low-priority queue (`thread_pool.rs:152-169`; blocks use high-priority); (3) validate in parallel on snapshot; (4) send `MutatorMessage::ExecutionProof { result, origin }`; (5) apply serially: `apply_execution_proof` + `update_store_snapshot()` + `origin.split()` → `P2pMessage::Accept/Ignore/Reject(gossip_id)`; (6) stale snapshots drain: second `Accept` becomes `Ignore(type known)` at apply time via type-new re-check.

Glossary: `Arc<T>` shared pointer; mpsc `Sender/Receiver` queue; `ArcSwap` non-blocking publish; `Result<Action>` = `Ok(Accept)/Ok(Ignore)/Err(reject)`; `origin.split()` separates gossip signal from API reply.

### 6.4 Carrier deep-dive — what crosses the channel

Types (`types/src/eip8025/containers.rs:148-169`): `ExecutionProofEnvelope { proof_data, proof_type: ProofType, beacon_block_root }` vs `SignedExecutionProofEnvelope { message: Hc<Envelope>, validator_index, signature }`. `Hc<T>` (`ssz/src/hc.rs:24`) = value + lazily-cached Merkle root; wrapped the way `SignedBeaconBlock` wraps its own, merkleized once for gossip-dedup keys and signing-root derivation (`containers.rs:158-161`).

P2P always delivers signed (auth gate needs signer + signature). Both carriers store the identical bare envelope; they differ only in surviving context: (A) `Accept(Arc<Signed>)` → mutator stores `signed.message`, keeps `validator_index` for `Seen.provers` marking, logging/metrics, event fan-out (cf. `handle_payload_bid`, `mutator.rs:2521-2526`), future k-of-n. (B) `Accept(Arc<Envelope>)` → task pre-extracts `.message`; mutator loses the author unless a second field is smuggled. Cost: 96 B signature + 8 B index vs ≤4 MiB proof; all `Arc` (clone = atomic bump, no copy). `Hc::clone` (`hc.rs:42-50`) carries the cached root. Locked: signed-carrier (§4.1).

### 6.5 Seen location — Store-owned sets (locked §4.2)

Spec `p2p-interface.md` defines `Seen`; Grandine precedent keeps gossip-seen inside `Store` (`seen_gossip_attesters/aggregators`, `store.rs:172-176`) so snapshot reads enforce "first wins" and `apply` marks atomically with the insert. Same for proofs: bounded `HashMap`/`HashSet` mirrors of `execution_proof_roots` + `execution_proof_provers`, checked in `validate_*`, marked in `apply_*`. Rejected: p2p-layer cache (splits dedup from Store, double-verify races).

### 6.6 wait_group — omit, Bid-style (locked §4.3)

`wait_group: W` is a test/sync latch (classic tasks like `AggregateAndProofTask:291` keep it). `PayloadBidTask` omits it — fire-and-forget gossip. Proofs are the same class (gossip-only, no Req/Resp, no head weight), so `MutatorMessage::ExecutionProof { result, origin }` with no latch. One-line re-add if a test needs it.

### 6.7 Origin — new ExecutionProofOrigin (locked §4.5)

Verified not to exist yet (grep 2026-09-07). Template `misc.rs:479-519`: `enum { Gossip(GossipId), Api(OneshotSender<...>) }` + `split()` / `gossip_id()`. Skeleton constructs only `Gossip` (`controller.rs:413` pattern); `Api` arm keeps HTTP-reply plumbing uniform. Per-type origin (not raw `GossipId`) keeps handling consistent.

### 6.8 Seen purpose — gossip's memory

Gossipsub delivers at-least-once over a mesh: same proof arrives 5-10× via peers, plus replays/spam. Without memory each duplicate redoes SSZ decode → BLS → `ProofEngine.verify` (ms-to-s): a free DoS amplifier. `Seen` = small "already handled" sets so duplicates die at a hash lookup before crypto. For proofs (`validate_execution_proof_gossip`): `execution_proof_roots[block] ∋ proof_root?` → IGNORE (replay); `execution_proof_provers ∋ (block, type, validator)?` → IGNORE (one attempt per prover per job — first costs a verify, next 1000 die at lookup); then cheap Store checks → IGNORE; then expensive checks (BLS, engine) → REJECT. Mark both sets after envelope auth, around engine verify — even engine-invalid attempts record the prover. Two maps because attackers vary whichever dimension is untracked: roots dedup by content, provers rate-limit by author.

### 6.9 Seen vs libp2p — two memories, two layers

libp2p gossipsub cache = network-level: dedups by message-ID (same bytes, different paths), IHAVE/IWANT. Understands nothing about signatures or fork choice. Consensus `Seen` = application-level: same content via different provers, same prover retrying new bytes, already-stored types. Flow: libp2p delivers bytes → consensus `validate_*` consults `Seen` + `Store` → verdict returns as `P2pMessage::Accept/Ignore/Reject(gossip_id)` (`mutator.rs:2530-2567`): forward / don't-forward-no-punish / don't-forward-and-downscore. `Seen` is the consensus side of that bridge, not a libp2p struct.
