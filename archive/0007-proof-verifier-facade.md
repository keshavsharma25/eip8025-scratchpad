# Scratchpad — ProofVerifier facade: why `Arc<dyn ProofEngine<P>>` could not compile and what we did instead (2026-09-08)

Base: Grandine `feature/sign-execution-proofs@9af0859`.
Branch: `proof-service-task-plumbing` @ `429841f` (stacked on `proof-service-state-shapes@215c2b1`).
Design doc: `design/docs/design/0003-execution-proof-service.md` (PR3 row).
Spec: `feat/simplify-eip8025` tip `6946894b0`, base `7d6bd46a0` (`proof-engine.md:26-40`).
Sequel to `0006-signed-carrier-vs-plumbing.md` (carrier/plumbing split) and `0004-proof-engine-discussions.md` (§1 stateless 3-method model).

This note records the one real design finding from PR3 implementation: the design doc's
`ProcessExecutionProofTask` sketch holds `Arc<dyn ProofEngine<P>>` (`0003 §Design`, task snippet),
but that type cannot exist. The committed code holds `Arc<dyn ProofVerifier<P>>` instead.
Everything below is the evidence trail and the rejected alternatives, so a future reviewer
(or the prover-role follow-up) does not re-litigate it.

## 1. The problem, exactly

PR1 (`proof_engine/src/engine.rs:20-41`) defines the spec-faithful twin of `ExecutionEngine`:

```rust
pub trait ProofEngine<P: Preset> {
    const IS_NULL: bool;
    fn verify_execution_proof(&self, execution_proof: ExecutionProof) -> bool;
    fn request_proofs(...) -> Result<H256, ProofEngineError>;
    fn get_proof(...) -> Result<ExecutionProof, ProofEngineError>;
}
```

PR3's task needs to own an engine handle across a thread-pool queue:

```rust
pub struct ProcessExecutionProofTask<P: Preset, W> {
    pub store_snapshot: Arc<Store<P, Storage<P>>>,
    pub proof_engine: Arc<dyn ProofEngine<P>>, // <- as sketched in 0003
    ...
}
```

`rustc` rejects this with `E0038: the trait ProofEngine is not dyn compatible ...
because it contains associated const IS_NULL`. This was proven, not assumed, with
standalone `rustc --edition 2021` reproductions during PR3:

- `trait T { const C: bool; fn f(&self) -> bool; }` + `fn g(x: &dyn T)` → `E0038`
  on both the parameter and every use site.
- Moving the const to a supertrait (`trait T: IsNull`) still fails: theSubtype
  bound poisons the subtrait's vtable for the same reason.
- `const C: bool where Self: Sized` (the usual "exclude the const from dyn" trick)
  fails differently: `generic const items are experimental (E0658, issue #113521)`.
  Stable Rust offers no way to keep a `const` on a `dyn`-compatible trait.

Rule of thumb: **methods can go in a vtable, associated consts (and generics) cannot.**
Any erased `dyn` handle must leave `IS_NULL` behind.

## 2. Why `IS_NULL` exists at all (and why we did not just delete it)

`IS_NULL` is the `ExecutionEngine` twin convention (`execution_engine/src/execution_engine.rs:22`,
`helper_functions/src/verifier.rs:22-23`). It encodes the opt-out invariant at the type level:

- `NullProofEngine::IS_NULL = true`, `verify` fail-closed (`false`), prover methods `Err(Unsupported)`.
- The task opens with the short-circuit (`tasks.rs:768`):
  `if proof_engine.is_null() { Ignore("proof engine not enabled") }`.
- Callers that are generic over the engine (block import, gossip validation) use
  `E::IS_NULL` / `V::IS_NULL` to skip work at compile time
  (e.g. `transition_functions/.../block_processing.rs`, `state_transition.rs`).

Deleting `IS_NULL` from `ProofEngine` would fix `dyn`-compatibility by amputation, but it would:
(a) fork the trait from its twin for no behavioral gain, (b) lose the compile-time
opt-out that every other engine/verifier in the tree provides, and (c) churn PR1
(which was already committed as `cc1733f` and green) to solve a PR3 storage problem.
The constraint adopted was therefore: **PR1 untouched; solve the erasure in `fork_choice_control`.**

## 3. The house pattern, surveyed (why nothing else in Grandine hits this)

Grep over the tree at PR3 tip shows the committed `Arc<dyn ProofVerifier<P>>`
(`tasks.rs:745`, tests, and one comment in `controller.rs:1003`) is the **only**
`Arc<dyn ...>` in Grandine. Everywhere else the codebase avoids erasure by staying generic:

- `Controller<P, E, A, W: Wait>` (`controller.rs:89`), `ThreadPool<P, E, W>`
  (`thread_pool.rs:41`), `BlockTask<P, E, W>` (`tasks.rs:76`),
  `ExecutionPayloadEnvelopeTask<P, E, W>` (`tasks.rs:639`) — all thread
  `E: ExecutionEngine<P> + Clone + Send + Sync + 'static`.
- `Verifier` (also `const IS_NULL`-carrying, `verifier.rs:22`) is always a generic
  method parameter (`V: Verifier` in `block_processor`/`Store::validate_*`); call sites
  pass concrete `MultiVerifier::default()` or `NullVerifier`. Never erased.
- `ExecutionPayloadBidTask<P, W>` (`tasks.rs:704`, introduced in `55b448b`
  "Handling gossip execution payload bid") carries no engine at all, so it never
  faces the question. It was originally even non-generic over the bid type
  (`Arc<SignedExecutionPayloadBid>`, no `<P>`); the proof task mirrors that
  minimalism with a non-generic envelope (`Arc<SignedExecutionProofEnvelope>`).

So the house answer to "trait with const + thread-pool task" is "add another generic."
We deliberately did not follow it here. §4 explains why.

## 4. Alternatives considered and rejected

### 4.1 Second engine generic `PE` on `Controller` / `ThreadPool` / tasks (the house answer)

```rust
pub struct Controller<P: Preset, E, PE, A, W: Wait> { ... proof_engine: PE ... }
enum LowPriorityTask<P: Preset, E, W> { ExecutionProof(ProcessExecutionProofTask<P, PE, W>) ... }
```

Pros: zero new traits, `E::IS_NULL`-style access (`PE::IS_NULL`) works, maximally consistent.
Cons (decisive): ripple for a stub. `Controller` is constructed in `new_internal`,
aliased in `specialized.rs` (`AdHocBenchController`, `BenchController`, `TestController`,
`TestExecutionEngine`), threaded through `mutator.rs`, `thread_pool.rs` (`Critical<P,E,W>`,
`Spawn<P,E,W>`), `binary_utils`, and every test helper (`helpers.rs:Context`,
`queries.rs` test constructors). A second engine generic touches all of them to carry
a handle that exactly one dead-code spawn path uses. For verifier-only skeleton scope
(0003 non-goals: no `client.rs`, no gossip wiring, weeks 9–11 for the real verifier),
that is the wrong trade: a large, review-obscuring diff whose only payoff is avoiding
a 10-line facade.

### 4.2 Delete `IS_NULL` from `ProofEngine`

Rejected per §2: breaks twin parity, loses the opt-out invariant, churns committed PR1.

### 4.3 Change `const IS_NULL` to `fn is_null(&self)` on the main trait

This would make `ProofEngine` itself dyn-compatible (methods are object-safe) and is
arguably the "cleanest" Rust. Rejected because it forks the trait surface from both
twins (`ExecutionEngine::IS_NULL`, `Verifier::IS_NULL` are consts; generic call sites
like `V::IS_NULL` in transition functions rely on const-ness). A verifier-only plumbing
PR is not the place to redesign a tree-wide convention; the cost would be borne by
every future diff that compares the two engines side by side.

### 4.4 Closed enum dispatch (`enum AnyProofEngine { Null(NullProofEngine), Mock(MockProofEngine), Client(Client) }`)

Pros: no traits, no generics, exhaustive matching.
Cons: closed world. `fork_choice_control` would depend on every engine implementation
including the future external-verifier `client.rs`, inverting the boundary 0003 deliberately
drew (proof_engine as a crate precisely so control does not know implementations).
Every new backend edits the enum and every match. Rejected.

### 4.5 Split the main trait now (object-safe `ProofVerifierPart` + `ProofProverPart`)

Over-engineering for the skeleton. The prover half (`request_proofs`/`get_proof`) is
reject-stubbed per spec permission (`proof-engine.md:39-40`) and never wired in
verifier-only scope. Splitting PR1's trait before any prover call site exists would
design for an unknown shape. The note in §6 records the split as the intended
prover-role evolution, not the skeleton.

### 4.6 `Arc<dyn Fn(...)>` closure capture

Loses identity (`is_null` vs `verify` become separate closures or a struct anyway),
harder to mock, harder to document against the spec protocol. Rejected as strictly
worse than a named facade.

## 5. Chosen: narrow object-safe facade, erased at the spawn boundary

Committed in `429841f` (`tasks.rs:727-741`, `controller.rs:1002-1019`):

```rust
// tasks.rs
pub trait ProofVerifier<P: Preset>: Send + Sync {
    fn is_null(&self) -> bool;
}
impl<P: Preset, E: ProofEngine<P> + Send + Sync> ProofVerifier<P> for E {
    fn is_null(&self) -> bool { E::IS_NULL }
}

pub struct ProcessExecutionProofTask<P: Preset, W> {
    pub store_snapshot: Arc<Store<P, Storage<P>>>,
    pub proof_engine: Arc<dyn ProofVerifier<P>>, // object-safe: methods only
    ...
}

// controller.rs
#[expect(dead_code)]
fn spawn_execution_proof_task<PE: ProofEngine<P> + Send + Sync + 'static>(
    &self,
    signed_proof: Arc<SignedExecutionProofEnvelope>,
    origin: ExecutionProofOrigin,
    proof_engine: Arc<PE>,
) {
    self.spawn(ProcessExecutionProofTask { ..., proof_engine, ... })
    //                              ^^^^^^^^^^^^ Arc<PE> -> Arc<dyn ProofVerifier<P>> coercion
}
```

Why this shape, point by point:

- **Object-safe by construction.** One method, no const, no generics → `dyn`-compatible.
  `Send + Sync` supertraits satisfy the thread-pool queue; `'static` is demanded only
  at the spawn boundary (queued tasks must own their data), not on the trait itself.
- **Blanket impl = zero per-backend code.** `Null`, `Mock`, and the future `Client`
  all get `ProofVerifier` free. The coercion needs the trait in scope at the spawn
  site and nothing else.
- **Erasure at the narrowest point.** `Controller`, `ThreadPool`, `LowPriorityTask`,
  `MutatorMessage`, and the mutator arm stay `(P, W)`-shaped — the exact bid-template
  shape PR3 cloned. Only the one spawn fn is generic, and it is private + dead-code
  until gossip wiring lands.
- **PR1 untouched.** The spec-faithful 3-method trait with `const IS_NULL` and
  `&`/`Arc`/`Mutex` forwards stands as committed. Reviewers comparing
  `proof_engine/src/engine.rs` against `proof-engine.md:26-40` see no accommodation hacks.
- **Stub-honest.** The facade today exposes only what the stub uses (`is_null`).
  `store_snapshot`/`signed_proof` are explicitly reserved (`let _ = ...`, `tasks.rs:763`)
  for the pipeline follow-up, which calls through the same facade in place.

## 6. Intended evolution (not skeleton scope)

- **Real pipeline:** `ProofVerifier` grows `verify_execution_proof(ExecutionProof) -> bool`
  (object-safe; already the only verifier-role method the gossip path calls per
  `p2p-interface.md` step 10). The `is_null` short-circuit stays first. No structural change.
- **Prover role (`client.rs`, weeks 9–11):** new `ProofProver<P>: Send + Sync`
  with `request_proofs` + `get_proof`, blanket-impl'd the same way, consumed by the
  prover path (API-driven, not gossip). Gossip code never imports it — verifier and
  prover windows stay separate, matching the spec's own baseline-vs-prover-role split
  (`proof-engine.md:26-40`).
- **If the tree ever makes `ExecutionEngine`/`Verifier` dyn-compatible** (e.g. by
  moving `IS_NULL` behind a method tree-wide), the facade collapses back to
  `Arc<dyn ProofEngine<P>>` in a ~15-line revert. The spawn boundary is the only
  coupling point, so the blast radius is known in advance.

## 7. Verification record

- `E0038` reproductions with standalone `rustc --edition 2021` (const on trait,
  const via supertrait, `where Self: Sized` const → `E0658`). No guessing.
- `cargo check -p fork_choice_control -p fork_choice_store -p types --features bls/blst,kzg_utils/blst` → pass.
- New tests `tasks.rs:1265-1295` (stub vs `Mock(true/false)`, `Null → Ignore`) → 2/2 pass
  (via `-p fork_choice_control -p fork_choice_store -p helper_functions` with the two
  pre-broken `test_resources` modules temp-disabled locally and restored; breakage is
  missing vendored `consensus-spec-tests` dirs, unrelated).
- PR2 store test still green; `fmt` clean; `clippy --cap-lints=warn` grep zero hits on
  `fork_choice_control/src` + `fork_choice_store/src` (full `-D warnings` clippy cannot
  link: pre-existing `dedicated_executor` `disallowed_types` error, unrelated).
- Repo-wide `Arc<dyn` grep returns only the facade lines — confirming §3's novelty claim.
- `git show 55b448b` confirms the bid-task lineage (minimal attestation-family reduction,
  originally non-generic); PR3 mirrors that minimalism rather than the later
  `wait_group`-carrying envelope/preferences tasks.
