# Execution Plan — 0002 Sign Execution Proofs (re-based on container stack)

Design doc: `/home/keshav-wsl/code/8025/design/docs/design/0002-sign-execution-proofs.md`
Grandine: `/home/keshav-wsl/code/8025/grandine`, branch `feature/sign-execution-proofs`
Status: fully executed 2026-09-01 — all 7 steps landed, all gates green. See "Execution log" (§ 9).
Re-based 2026-09-03 onto the rewritten container stack (base `366fabc`); § 10 plans the envelope re-target (2026-09-04).

## Resolved decisions

| #   | Decision                                                                                                                                                                                                                   |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Rebase `feature/sign-execution-proofs` (currently `= eaf220e`, clean) onto `feature/eip8025-payload-binding` (`368f36e`); stack is linear on that base — *superseded: base was rewritten; re-based onto `366fabc` (§ 10.1)* |
| 2   | `DOMAIN_EXECUTION_PROOF = 0x0F000000` (consensus-specs; simplify branch `tau-lepton/feat/simplify-eip8025` = tie-breaker only, **not** adopted as baseline); added by **this** workstream in `types/src/eip8025/consts.rs` — *domain value stands; "not adopted" superseded, the rewritten base follows the simplify shape (§ 10.1)* |
| 3   | Sign the stack's non-generic `ExecutionProof`; envelope shape left to upstream — *superseded by R4 (§ 10)*                                                                                                                  |
| 4   | 4 commits, `feat:`/`test:` split                                                                                                                                                                                           |
| 5   | Test suite a–e, Minimal preset                                                                                                                                                                                             |
| 6   | Doc 0002 full re-baseline                                                                                                                                                                                                  |

## Key facts established by exploration

- Stack (linear, all off `eaf220e`, present locally and on `upstream` remote):
  1. `feature/eip8025-progressive-merkleization` = `e6694f3` "Implement `merkleize_progressive`" (ssz crate)
  2. `feature/eip8025-progressive-byte-list` = `c89b7d3` "Add `ProggressiveByteList`" (ssz/src/progressive_byte_list.rs)
  3. `feature/eip8025-proof-containers` = `eda4b27` "Add EIP-8025 execution proof containers" (types/src/eip8025/)
  4. `feature/eip8025-payload-binding` = `368f36e` "Add EIP-8025 payload binding root" (tip)
  — *superseded by the 2026-09-03 rewrite: see § 10.1 for the new base and commit hashes.*
- Module is `types::eip8025` (no underscore); files: `consts.rs`, `containers.rs`, `primitives.rs`, `container_impls.rs`, `error.rs`, `tests.rs`.
- `ExecutionProof` is **non-generic**: `proof_data: ProofData`, `proof_type: ProofType` (= u8), `public_input: PublicInput { new_payload_request_root: H256 }` — *shape evolved: `PublicInput` now has 4 fields (§ 10.1).*
- `ProofData` wraps `ProgressiveByteList`; `MAX_PROOF_SIZE: usize = 0x0040_0000` (4 MiB) enforced at runtime via `TryFrom<Vec<u8>>` + decode/serde; `MAX_SIGNED_EXECUTION_PROOF_SIZE = 108 + 37 + MAX_PROOF_SIZE` — *renamed to `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` on the rewritten base.*
- `SignedExecutionProof.message` is `Hc<ExecutionProof>`; `Hc<T>: SszHash` delegates/caches (ssz/src/hc.rs:107), so signing `Hc<ExecutionProof>` ≡ signing `ExecutionProof` — *superseded: container is now `SignedExecutionProofEnvelope { message: Hc<ExecutionProofEnvelope>, … }` (§ 10.1).*
- `DOMAIN_EXECUTION_PROOF` is deliberately absent from the stack's `consts.rs` ("signing out of scope for this milestone") — this workstream adds it.
- `DomainType = H32` (types/src/phase0/primitives.rs:8); const pattern: `pub const DOMAIN_X: DomainType = H32(hex!("0F000000"));` (phase0/consts.rs:14).
- `SignatureKind` never exhaustively matched (constructed + Display only) — additive variant safe; alphabetical slot after `ExecutionPayloadEnvelope`; Display convention `"<thing> signature"`.
- `SignForSingleForkAtSlot` trait (helper_functions/src/signing.rs:172): requires `SszHash` (provided by `#[derive(Ssz)]`), consts `DOMAIN_TYPE` + `SIGNATURE_KIND`, default `signing_root`/`sign`/`verify`. Default `signing_root` = domain at `compute_epoch_at_slot(slot)` via `accessors::get_domain` (reads `state.fork()`, accessors.rs:551) + `misc::compute_signing_root(self, domain)`.
- `H256` impl of the trait at signing.rs:452 is the pattern to mirror.
- `compute_domain` takes `(config, domain_type, Option<Version>, Option<H256>)` (misc.rs:207); fork versions are `Config` fields (`config.genesis_fork_version`), not `Preset` items — `Minimal::genesis_fork_version()` does not exist.
- Simplify branch (`tau-lepton/feat/simplify-eip8025`): renames the signed message to `ExecutionProofEnvelope { proof_data, proof_type, beacon_block_root }` with 4-field progressive `PublicInput` and progressive `NewPayloadRequest`. *Was "in-works, not adopted" — the rewritten base now adopts it (§ 10.1).*
- Stack consts.rs cites consensus-specs at `a08d8a6e2b45f0b8c0d379abc15583427c643689`; the rewritten base cites `7d6bd46a015a7dd316c5df855bd89e57c4aa6700` (types/src/eip8025/consts.rs header) — new edits cite that.
- Commit style evidence: Grandine develop mixes plain imperative and `feat:`/`test:` prefixes; user chose conventional prefixes.
- Env: rustc/cargo 1.95.0, edition 2024. zsh gotcha: `echo ===` fails — quote separators.

## 1. Rebase

`git rebase feature/eip8025-payload-binding feature/sign-execution-proofs` — branch has no own commits, so this is a fast-forward to `368f36e`.

Note: `feature/sign-execution-proofs` has no upstream set. Plan assumes local-only commits; pushing (`-u` to `upstream`) is a separate, explicitly-decided step.

## 2. `feat: Add DOMAIN_EXECUTION_PROOF` — `types/src/eip8025/consts.rs` (file from stack branch)

The commit lands on `feature/sign-execution-proofs` on top of the rebase — not on
`feature/eip8025-payload-binding`, which is a published stack branch.

Replace the "deliberately absent" paragraph in the module doc — simply delete it: the module
doc's pre-existing "Where consensus-specs and the EIP text disagree, these follow consensus-specs"
line already covers the divergence, so no replacement prose is needed. Keep the existing
`[EIP-8025]: …a08d8a6e2` citation untouched. Append (one-line doc; review decision A3 trimmed the
longer divergence prose originally planned here):

```rust
/// The domain used for signing execution proofs.
pub const DOMAIN_EXECUTION_PROOF: DomainType = H32(hex!("0F000000"));
```

Add `hex` + `H32`/`DomainType` imports as needed.

## 3. `feat: Add SignatureKind::ExecutionProof` — `helper_functions/src/error.rs`

After `ExecutionPayloadEnvelope` (line 99), alphabetical:

```rust
#[display("execution proof signature")]
ExecutionProof,
```

No exhaustive matches exist; additive + Display-only.

## 4. `feat: Implement signing for ExecutionProof` — `helper_functions/src/signing.rs`

- Imports: add `eip8025::{consts::DOMAIN_EXECUTION_PROOF, containers::ExecutionProof}` to the `types` use block (after `deneb`, before `electra`).
- Registry entry after the `ExecutionPayloadEnvelope` impl (line 478–485), mirroring the `H256`/`SignForSingleForkAtSlot` pattern (line 452):

```rust
/// <https://github.com/ethereum/consensus-specs/blob/321eca5b71049fcac6c63c2d956e5c5d7b60d689/specs/_features/eip8025/prover.md#new-get_execution_proof_envelope_signature>
impl<P: Preset> SignForSingleForkAtSlot<P> for ExecutionProof {
    const DOMAIN_TYPE: DomainType = DOMAIN_EXECUTION_PROOF;
    const SIGNATURE_KIND: SignatureKind = SignatureKind::ExecutionProof;
}
```

Sign/verify/signing_root come from the trait default. Signing `ExecutionProof` ≡ signing `SignedExecutionProof.message: Hc<ExecutionProof>` because `Hc: SszHash` delegates (ssz/src/hc.rs:107) — no extra impl.

## 5. `test: add execution proof signing tests` — `helper_functions/src/signing/tests.rs`

Executed: tests live in a **separate child module file** (`signing/tests.rs`, wired via
`#[cfg(test)] mod tests;` at the bottom of `signing.rs`), not inline — matching upstream's
no-tests-in-source-file pattern. This was the user's relocation decision at review time.
Constants mirror the stack's tests.rs (`NEW_PAYLOAD_REQUEST_ROOT = 0x000102…1e1f`, `PROOF_TYPE = 7`,
`test_bytes(n)` = 0..n; `test_signature()` helper **dropped** — dead code under `-D warnings`).

State: `Config::minimal()` + `Phase0BeaconState::<Minimal>::default()` with `fork = Fork { previous_version: config.genesis_fork_version, current_version: config.genesis_fork_version, epoch: 0 }` (`get_domain` reads `state.fork()`, accessors.rs:551). `config.genesis_fork_version` is `0x00000001` under minimal; see `test_compute_domain` (misc.rs:1051) for the existing pattern.

Suite:

- **(a) formula**: `proof.signing_root(config, &state, slot)` == `compute_signing_root(proof.hash_tree_root(), compute_domain(config, DOMAIN_EXECUTION_PROOF, Some(fork_version), Some(state.genesis_validators_root())))`, where `fork_version` is `state.fork().previous_version` or `state.fork().current_version` chosen by `compute_epoch_at_slot::<Minimal>(slot) < state.fork().epoch` — mirroring `accessors::get_domain` (accessors.rs:551). `compute_domain` takes `Option<Version>`, not an epoch, so the epoch itself never appears as the domain's fork version (misc.rs:207). Anchor the domain independently of `get_domain` by also pinning the literal 32-byte domain hex (computed out-of-band) the way `test_compute_domain` does (misc.rs:1059 pins `0100000018ae4ccb…`). Run two fork-selection cases: the base state (`fork.epoch = 0` → `current_version` path) and a state with `fork.epoch > 0` plus a slot from an earlier epoch (→ `previous_version` path; e.g. `fork.epoch = 1`, slot `0`, `previous_version = config.genesis_fork_version`).
- **(b) pinned vector**: len-100 proof (object root `9727e301e9d88ac931277369c166b74568fd7b5172417944a7245a43a08cfbf5` — stack's independently-computed reference) → full signing-root hex pinned with provenance comment. Compute the pinned value out-of-band (hand/script merkleization, not by the code under test), with the same caveat as the stack's roots (not pyspec-cross-checked).
- **(c) round-trip**: `sign` with `SecretKey` replicating the verifier.rs:473 `secret_key()` helper (`b"????????????????????????????????"` → `SecretKeyBytes` → `.try_conv()`; it is a private fn inside verifier.rs's test module — copy the pattern, don't import); `sign` returns `bls::Signature`, `verify` takes `SignatureBytes` — convert with `.into()` (verifier.rs:466) before calling `verify` with a matching `Arc<PublicKey>` → `Ok`
- **(d) negative**: tampered message → `Error::SignatureInvalid(SignatureKind::ExecutionProof)`, Display = `"execution proof signature"`
- **(e) Hc parity**: `SignedExecutionProof { message: Hc::from(proof.clone()), … }` — `signed.message.hash_tree_root() == proof.hash_tree_root()`, and signature over bare proof verifies

## 6. Doc 0002 full re-baseline — design repo

`/home/keshav-wsl/code/8025/design`, branch `docs/0001-sign-execution-proofs`, file `docs/design/0002-sign-execution-proofs.md`:

- **Front-matter**: keep pins; add `grandine_stack_tip: 368f36e` + stacked-branch list.
- **Context**: mark `0x0F` resolved/adopted (EIP `0x0D` stale — collides with the spec's Gloas `DOMAIN_PROPOSER_PREFERENCES`, not present in Grandine's `gloas/consts.rs` at this pin; simplify branch cited as tie-breaker only); update the provisional table rows that the stack settled (1-field `PublicInput` as-implemented, `ProgressiveByteList`, 4 MiB).
- **Design**: `types/src/eip_8025/` → `types/src/eip8025/`; fix impl snippet to non-generic `ExecutionProof`; flip `DOMAIN_EXECUTION_PROOF` ownership from [0004] to this workstream; note `Hc<ExecutionProof>` message wrapper.
- **Implementation/testing**: unblock step 2 (containers exist on the stack); record the 4-commit plan + test suite.
- **Blockers**: rewrite — containers landed on stacked branches (not yet merged upstream); domain value adopted pending upstream convergence; one-line note that the in-works simplify branch moves the signed message to `ExecutionProofEnvelope` (not adopted; impl retargets with a one-line change).
- **Open questions**: trim resolved items.

## 7. Gates

Amended during execution (see "Resolved decisions" addendum below): all `-p types`/`-p helper_functions` invocations need `--features bls/blst` because the workspace maps `bls = { path = 'bls' }` without a BLS backend (only the `grandine` binary enables one via its default features). Clippy additionally needs three allow-flags for pre-existing, upstream-owned failures (verified identical on the untouched stack base; do **not** fix root `Cargo.toml` in this fork):

1. `cargo check -p helper_functions --all-targets --features bls/blst` (`--all-targets` is what compiles test code)
2. `cargo test -p helper_functions --features bls/blst` (full crate; new signing module included)
3. `cargo test -p types --features bls/blst` (consts.rs + module doc change lives in types)
4. `cargo fmt --all -- --check`
5. `cargo clippy -p helper_functions --features bls/blst -- -D warnings -A clippy::lint_groups_priority -A unfulfilled_lint_expectations -A unexpected_cfgs` (plus `-p types --features bls/blst` for the consts change)

The three clippy allows map to pre-existing issues in upstream-owned code:

| Allow | Location | Cause |
| --- | --- | --- |
| `-A clippy::lint_groups_priority` | root `Cargo.toml` lint table | `deprecated_safe` (a lint group as of newer rustc) and group member `unstable_name_collisions` both at implicit priority 0 |
| `-A unfulfilled_lint_expectations` | `bls-core` | 4 `#[expect(clippy::module_name_repetitions)]` that no longer fire under rustc/clippy 1.95 |
| `-A unexpected_cfgs` | `bls-blst` | macro-generated `cfg(feature = 'dev')` unknown to the toolchain |

Caveat: the allow-flags apply to this workstream's code too (e.g. an unfulfilled `#[expect]` in new code would not be caught); acceptable because the additions use neither `#[expect]` nor custom cfgs.

## 8. Resolved decisions addendum (execution-time)

| #   | Decision                                                                                                                                                        |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1  | Do not touch root `Cargo.toml` (fork must stay diff-clean vs upstream); solve gate failures via CLI flags only, recorded in § 7                                 |
| A2  | All `-p types` / `-p helper_functions` gates require `--features bls/blst` (workspace `bls` dep has no backend feature; only `grandine` binary enables it)       |
| A3  | User review trimmed the `DOMAIN_EXECUTION_PROOF` doc prose to one line (§ 2); divergence rationale stays in design doc 0002, not in code comments               |

## 9. Execution log (final state, 2026-09-01)

Grandine `feature/sign-execution-proofs` — 4 commits on the stack tip `368f36e` (pre-rewrite hashes; superseded by § 10.1):

| Commit    | Subject                                    | Notes |
| --------- | ------------------------------------------ | ----- |
| `81ea62d` | feat: Add DOMAIN_EXECUTION_PROOF           | amended via interactive rebase (was `72ab5a8`); one-line doc per A3 |
| `c3086ba` | feat: Add SignatureKind::ExecutionProof    | replayed onto amended base (was `112d1fe`) |
| `fe652f1` | feat: Implement signing for ExecutionProof | non-generic impl, trait defaults |
| `4cae85e` | test: add execution proof signing tests    | suite a–e; tests relocated to `signing/tests.rs` (was `dc91995` inline) |

Design repo (`docs/0001-sign-execution-proofs`, tracks origin — commit is **local until pushed**):
`d7d1f62` "docs: re-baseline 0002 sign execution proofs" (+100/−59 in `docs/design/0002-sign-execution-proofs.md`).

Gates — all green against final HEAD `4cae85e`:

| Gate | Command | Result |
| ---- | ------- | ------ |
| check | `cargo check -p helper_functions --features bls/blst` | pass |
| tests | `cargo test -p helper_functions --features bls/blst` | 412 passed |
| tests | `cargo test -p types --features bls/blst` | 57,141 + 1 passed |
| fmt | `cargo fmt --all -- --check` | pass |
| clippy | `cargo clippy -p helper_functions -p types --features bls/blst -- -D warnings -A clippy::lint_groups_priority -A unfulfilled_lint_expectations -A unexpected_cfgs` | pass |

Pins landed in tests (out-of-band, sha256-script computed, not pyspec-cross-checked): domain
`0f00000018ae4ccbda9538839d79bb18ca09e23e24ae8c1550f56cbb3d84b053`, signing root
`437b6ffa8ebcc64190a33c510af5a1ed19f649eff7bca884bc6dac505548f160`, len-100 object root
`9727e301e9d88ac931277369c166b74568fd7b5172417944a7245a43a08cfbf5`.

Outstanding (user's call): pushing both branches; hands-on interactive-rebase walkthrough queued as a post-completion item.

## 10. Re-baseline #2 — signed object re-targets to `ExecutionProofEnvelope` (2026-09-03/04)

Trigger: peer force-pushed a rewritten `feature/eip8025-payload-binding` that adopts the
simplify-branch container shape wholesale; this branch was re-based onto it. The old stacked
branches (§ "Key facts" 1–4) are superseded history.

| #   | Decision                                                                                                                                                        |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R4  | Signed object = `ExecutionProofEnvelope`. Resolved by the spec itself, no longer an open question for Charles: `feat/simplify-eip8025`'s `get_execution_proof_envelope_signature` signs the envelope. Supersedes decision 3 |
| R5  | Land as **one follow-up commit** on top (pivot stays visible for PR review; no history rewrite needed)                                                          |

### 10.1 Established facts

- Rebase completed 2026-09-03, all 4 commits replayed cleanly onto base `366fabc` ("Fix EIP-8025 consensus-specs link"): `f335537` (domain) → `372b368` (SignatureKind) → `0c47ad5` (impl) → `07c26c6` (tests). Only conflict: `consts.rs` (both sides added imports; resolved by authorship — both import sets kept, the user's `DOMAIN_EXECUTION_PROOF` kept, peer's rename + removals honored).
- Base containers (`types/src/eip8025/containers.rs`) now match `feat/simplify-eip8025` exactly: 4-field progressive `PublicInput` (`new_payload_request_root`, `successful_validation`, `chain_id: u64`, `schema_id: u16`), bare `ExecutionProof`, gossiped `ExecutionProofEnvelope { proof_data, proof_type, beacon_block_root }`, `SignedExecutionProofEnvelope { message: Hc<ExecutionProofEnvelope>, validator_index, signature }`.
- Spec source of truth: local `/home/keshav-wsl/code/8025/consensus-specs` @ `feat/simplify-eip8025` (tip `6946894b0`, already checked out; matches ethereum/consensus-specs master):
  - `get_execution_proof_envelope_signature(state, proof_envelope, privkey)`: `compute_signing_root(proof_envelope, domain)`, domain epoch = `compute_epoch_at_slot(state.slot)` (prover.md). Verifier side identical (beacon-chain.md).
  - `PublicInput` is **proof-engine-facing only**: prover validates `chain_id == DEPOSIT_CHAIN_ID` / `schema_id == STATELESS_INPUT_SCHEMA_ID` locally, then drops it (`get_signed_execution_proof_envelope`); the bare `ExecutionProof` is reconstructed by `get_execution_proof` for engine submission. It never travels and never enters the signing root.
  - Constants unchanged: `DOMAIN_EXECUTION_PROOF = 0x0F000000`, `MAX_PROOF_SIZE = 4 MiB`, `STATELESS_INPUT_SCHEMA_ID = 0x1501`; `DEPOSIT_CHAIN_ID`: minimal = 5, mainnet = 1 (`configs/minimal.yaml` / grandine `Config::minimal().deposit_chain_id`).
- User's impl (`0c47ad5`) follows the superseded Aug-10 spec (`321eca5`, `get_execution_proof_signature` over the bare proof): the doc-comment anchor says envelope while the commit pins the old heading. Re-target per R4.
- Uncommitted WIP on the branch (partial mechanical test fix) does **not** compile: `tests.rs:40` can't resolve `ExecutionProofEnvelope` (import line has `ExecutionProof` + `SignedExecutionProofEnvelope` but not it); behind it: `test_proof` return type vs body mismatch and dead `tampered_proof.public_input`. WIP is absorbed into the follow-up commit.
- Compile gate today shows exactly 4 pre-existing `spec_tests.rs` errors (2 `cryptography-spec-tests` macro panics + 2 E0425 cascade from the failed expansion) — proven not ours (empty diff at base). Note: name-resolution errors abort before typeck, so they mask type errors elsewhere; judge compile state only after they're accounted for.
- Pinned vectors re-verified out-of-band: domain `0f00000018ae4ccbda9538839d79bb18ca09e23e24ae8c1550f56cbb3d84b053` == `compute_domain` for the genesis state (fork_version = minimal genesis `0x00000001`, gvr = 0; cf. `test_compute_domain` misc.rs:1059); old signing root `437b6ffa…` == sha256(bare-proof-root ++ domain). Formula test (a) survives unchanged; only object-root pins change.
- New envelope pins (out-of-band, same independent-computation caveat as § 9): object root = `024c73d1626ebced9a5e284ec4bd9d4f893e36609dd302c6e0a3b41d84ba396d` (types-suite cross-checked envelope root @ len 100, `types/src/eip8025/tests.rs:120`), signing root = `cde2fa0216202a493abf3206102c5367ed597806a7387cef9dc8e21e341fbc18` = sha256(object_root ++ domain). Reusing the types-suite sample values (`BEACON_BLOCK_ROOT = 0x2021…3f`, `PROOF_TYPE = 7`, `test_bytes(100)`) is what makes the cross-checked root reusable — do not invent new sample values.

### 10.2 Changes

**`helper_functions/src/signing.rs`**

1. Import `eip8025::containers::ExecutionProofEnvelope` (drop `ExecutionProof`); keep the uncommitted rustfmt import reorder.
2. Re-target the impl and re-pin the doc link to the base's cited spec commit `7d6bd46a…` (anchor finally matches the heading):

```rust
/// <https://github.com/ethereum/consensus-specs/blob/7d6bd46a015a7dd316c5df855bd89e57c4aa6700/specs/_features/eip8025/prover.md#new-get_execution_proof_envelope_signature>
impl<P: Preset> SignForSingleForkAtSlot<P> for ExecutionProofEnvelope {
    const DOMAIN_TYPE: DomainType = DOMAIN_EXECUTION_PROOF;
    const SIGNATURE_KIND: SignatureKind = SignatureKind::ExecutionProof;
}
```

**`helper_functions/src/signing/tests.rs`** — rewrite around the envelope:

- Constants: `NEW_PAYLOAD_REQUEST_ROOT` → `BEACON_BLOCK_ROOT = 0x2021…3f` (mirroring `types/src/eip8025/tests.rs`); keep `PROOF_TYPE = 7`, `PROOF_DATA_LENGTH = 100`. Drop the `PublicInput` import — it leaves this module entirely (spec: never signed).
- `test_proof()` → envelope struct literal `ExecutionProofEnvelope { proof_data, proof_type, beacon_block_root: BEACON_BLOCK_ROOT }`.
- **(a) formula**: logic unchanged, object = envelope; the pinned domain survives; fork-boundary case keeps its value (fork selection is orthogonal to object choice).
- **(b) pinned vector**: object root `024c73d1…` + signing root `cde2fa02…` with provenance comment (types-suite cross-check + out-of-band sha256, not pyspec-cross-checked).
- **(c) round-trip**: sign/verify the envelope (trait defaults).
- **(d) negative**: tamper `beacon_block_root = H256::repeat_byte(0xff)`; keep `Error::SignatureInvalid(SignatureKind::ExecutionProof)` + Display assertions.
- **(e) now true**: sign envelope → wrap in `SignedExecutionProofEnvelope { message: Hc::from(envelope.clone()), validator_index: 0, signature }` → assert `signed.message.hash_tree_root() == envelope.hash_tree_root()`; mirrors `get_signed_execution_proof_envelope`.
- New: envelope root ≠ bare-proof root for identical proof data (mirrors `envelope_and_proof_roots_differ` in the types suite) — encodes "we sign the envelope" against regression; comment cites `get_execution_proof` for where `PublicInput` actually lives.
- If bare-proof tests ever return: spec-correct values are `chain_id = config.deposit_chain_id` (minimal = 5) and `schema_id = STATELESS_INPUT_SCHEMA_ID` (`0x1501`); the types-suite reference used Sepolia's `11_155_111` as its sample chain id.

### 10.3 Gates

Per § 7 (A2 flags apply). Sequence:

1. `cargo check -p helper_functions --all-targets --features bls/blst` — must show **only** the 4 pre-existing `spec_tests.rs` errors.
2. `cargo test -p helper_functions --features bls/blst signing` — all green.
3. `cargo fmt --all -- --check`; clippy per § 7 flags.
4. `git push --force-with-lease origin feature/sign-execution-proofs` (remote still at pre-rebase tip `4cae85e`).

### 10.4 Terminology (this session's Q)

`PublicInput { … }` is a **struct literal** (Rust Reference: *struct expression*); it produces a **value** of the type. "Instantiate" / "an instance of `PublicInput`" is fine colloquially; Rust docs say *value*. "Object" is reserved for trait objects and SSZ "object root" jargon.
