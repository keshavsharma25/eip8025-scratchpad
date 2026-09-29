# `SszNewPayloadRequest` → `NewPayloadRequest` (spec #5619)

Branch: `feature/eip8025-payload-binding` (`366fabc`, base `9dc532a`).
Spec: [#5619](https://github.com/ethereum/consensus-specs/pull/5619), from [#5566#discussion_r3875310633](https://github.com/ethereum/consensus-specs/pull/5566#discussion_r3875310633).
Precedent: [ChainSafe/lodestar#10051](https://github.com/ChainSafe/lodestar/pull/10051) (nflaig, merged 2026-09-09).

#5619 is a fork-wide change: the engine-API request type became a first-class beacon-chain container, versioned per fork (bellatrix 1 field, deneb 3, electra 4, gloas 4 progressive). Only the Gloas shape is needed by EIP-8025; the rest is spec-structure parity. Part A is the 8025 change, Part B is the structural mirror.

## Part A — what EIP-8025 needs (Gloas)

Rename + relocate the existing container `eip8025` → `gloas`. Fields, order and `active = [1; 4]` are unchanged, so all roots are unchanged.

### A1. `types/src/gloas/containers.rs`

```diff
-    deneb::primitives::{Blob, KzgCommitment, KzgProof},
+    deneb::primitives::{Blob, KzgCommitment, KzgProof, VersionedHash},
```

```diff
 pub struct ExecutionRequests<P: Preset> {
     pub deposits: ProgressiveList<DepositRequest>,
     pub withdrawals: ProgressiveList<WithdrawalRequest>,
     pub consolidations: ProgressiveList<ConsolidationRequest>,
     pub builder_deposits: ProgressiveList<BuilderDepositRequest>,
     pub builder_exits: ProgressiveList<BuilderExitRequest>,
     #[ssz(skip)]
     #[serde(skip)]
     pub phantom: PhantomData<P>,
 }
 
+/// The `NewPayloadRequest` whose execution a proof certifies.
+///
+/// `hash_tree_root` of this container is
+/// `public_input.new_payload_request_root`, the only link between a
+/// proof and the payload it certifies.
+///
+/// Mirrors the `NewPayloadRequest` progressive container in
+/// `specs/gloas/beacon-chain.md`. The root is preset-independent:
+/// every list that could carry a limit into it is progressive, and
+/// the preset-derived bounds that do reach it are equal across
+/// presets, which `eip8025::container_impls` asserts.
+#[derive(Clone, PartialEq, Eq, Default, Debug, Deserialize, Serialize, Ssz)]
+#[serde(bound = "", deny_unknown_fields)]
+#[ssz(stable(active = [1; 4]))]
+pub struct NewPayloadRequest<P: Preset> {
+    pub execution_payload: ExecutionPayload<P>,
+    // consensus-specs bounds this at
+    // `MAX_BLOB_COMMITMENTS_PER_BLOCK`, which is what
+    // `MaxBlobCommitmentsPerBlock` is.
+    pub versioned_hashes: ContiguousList<VersionedHash, P::MaxBlobCommitmentsPerBlock>,
+    pub parent_beacon_block_root: H256,
+    pub execution_requests: ExecutionRequests<P>,
+}
+
 #[derive(Clone, Copy, PartialEq, Eq, Debug, Deserialize, Serialize, Ssz)]
 #[serde(bound = "", deny_unknown_fields)]
 pub struct ForkChoiceNode {
```

### A2. `types/src/eip8025/containers.rs`

```diff
 use ssz::{
-    ContiguousList, Hc, ProgressiveByteList, ReadError, Size, Ssz, SszHash, SszRead, SszSize,
-    SszWrite, WriteError,
+    Hc, ProgressiveByteList, ReadError, Size, Ssz, SszHash, SszRead, SszSize, SszWrite,
+    WriteError,
 };
 
 use crate::{
-    deneb::primitives::VersionedHash,
     eip8025::{
         consts::MAX_PROOF_SIZE,
         primitives::{MaxProofSize, ProofType},
     },
-    gloas::containers::{ExecutionPayload, ExecutionRequests},
     phase0::primitives::ValidatorIndex,
-    preset::Preset,
 };
```

```diff
-/// The `SSZNewPayloadRequest` whose execution a proof certifies.
-///
-/// `hash_tree_root` of this container is
-/// `public_input.new_payload_request_root`, the only link between a
-/// proof and the payload it certifies.
-///
-/// A `ProgressiveContainer` in consensus-specs, built from the Gloas
-/// `ExecutionPayload` and `ExecutionRequests`. Grandine's Rust name
-/// follows the spec's `SSZNewPayloadRequest`; the `SSZ` prefix
-/// distinguishes it from the Engine API request of the same name,
-/// which is not an SSZ container at all.
-///
-/// The root is preset-independent. Under Gloas every list that could
-/// carry a limit into it is progressive, and the preset-derived
-/// bounds that do reach it are equal across presets, which
-/// `container_impls` asserts.
-#[derive(Clone, PartialEq, Eq, Default, Debug, Deserialize, Serialize, Ssz)]
-#[serde(bound = "", deny_unknown_fields)]
-#[ssz(stable(active = [1; 4]))]
-pub struct SszNewPayloadRequest<P: Preset> {
-    pub execution_payload: ExecutionPayload<P>,
-    // consensus-specs bounds this at
-    // `MAX_BLOB_COMMITMENTS_PER_BLOCK`, which is what
-    // `MaxBlobCommitmentsPerBlock` is.
-    pub versioned_hashes: ContiguousList<VersionedHash, P::MaxBlobCommitmentsPerBlock>,
-    pub parent_beacon_block_root: H256,
-    pub execution_requests: ExecutionRequests<P>,
-}
```

### A3. `types/src/eip8025/container_impls.rs`

```diff
 use crate::{
     combined::{ExecutionPayload as CombinedExecutionPayload, ExecutionPayloadParams},
-    eip8025::{containers::SszNewPayloadRequest, error::PayloadBindingError},
+    eip8025::error::PayloadBindingError,
+    gloas::containers::NewPayloadRequest,
     preset::{Mainnet, Minimal, Preset},
 };
```

```diff
-// Most of what `SSZNewPayloadRequest` contains carries no limit into
+// Most of what `NewPayloadRequest` contains carries no limit into
 // its root: `transactions`, `withdrawals` and every field of
```

```diff
-impl<P: Preset> SszNewPayloadRequest<P> {
-    /// Reconstructs the spec's `SSZNewPayloadRequest` from the pair
+impl<P: Preset> NewPayloadRequest<P> {
+    /// Reconstructs the spec's `NewPayloadRequest` from the pair
     /// Grandine already holds at the `notify_new_payload` boundary.
```

Body of `new()` unchanged.

### A4. `types/src/eip8025/error.rs`

```diff
     #[error(
         "execution payload of phase {phase} cannot be bound; \
-         EIP-8025 binds the Gloas shape of SSZNewPayloadRequest"
+         EIP-8025 binds the Gloas shape of NewPayloadRequest"
     )]
     PayloadPhaseNotSupported { phase: Phase },
     #[error(
         "execution payload params without Gloas execution requests cannot be bound; \
-         EIP-8025 binds the Gloas shape of SSZNewPayloadRequest"
+         EIP-8025 binds the Gloas shape of NewPayloadRequest"
     )]
     ExecutionRequestsNotGloas,
```

### A5. `types/src/preset.rs`

```diff
     type MaxBlobCommitmentsPerBlock: MerkleElements<Blob<Self>>
         + MerkleElements<Cell<Self>>
         + MerkleElements<KzgCommitment>
-        // Needed for `SszNewPayloadRequest.versioned_hashes`, which
+        // Needed for `NewPayloadRequest.versioned_hashes`, which
         // is a list of `VersionedHash`.
         + MerkleElements<VersionedHash>
```

### A6. `types/src/eip8025/tests.rs`

```diff
         containers::{
             ExecutionProof, ExecutionProofEnvelope, ProofData, PublicInput,
-            SignedExecutionProofEnvelope, SszNewPayloadRequest,
+            SignedExecutionProofEnvelope,
         },
         error::PayloadBindingError,
         primitives::ProofType,
     },
-    gloas::containers::{ExecutionPayload, ExecutionRequests},
+    gloas::containers::{ExecutionPayload, ExecutionRequests, NewPayloadRequest},
```

Replace `SszNewPayloadRequest` → `NewPayloadRequest` at lines 395, 456, 468, 501, 516, 534, 555, 569, 570. Test bodies and expected roots unchanged.

### A7. `types/src/lib.rs`

No change. Both module trees already exist.

## Part B — structural parity with the spec (per-fork containers)

Lodestar added all of these in #10051 to match the spec and its `ssz_static` vectors. Add only if Grandine wants to mirror the fork ladder and track the vectors; independent of EIP-8025.

### B1. `types/src/deneb/containers.rs`

```diff
-    deneb::primitives::{Blob, BlobCommitmentInclusionProof, BlobIndex, KzgCommitment, KzgProof},
+    deneb::primitives::{
+        Blob, BlobCommitmentInclusionProof, BlobIndex, KzgCommitment, KzgProof, VersionedHash,
+    },
```

```diff
 pub struct ExecutionPayload<P: Preset> {
     ...
     pub excess_blob_gas: Gas,
 }
 
+/// Mirrors `VersionedHashes` from `deneb/beacon-chain.md`.
+pub type VersionedHashes<P> =
+    ContiguousList<VersionedHash, <P as Preset>::MaxBlobCommitmentsPerBlock>;
+
+/// Mirrors `NewPayloadRequest` from `deneb/beacon-chain.md`.
+#[derive(Clone, PartialEq, Eq, Default, Debug, Deserialize, Serialize, Ssz)]
+#[serde(bound = "", deny_unknown_fields)]
+pub struct NewPayloadRequest<P: Preset> {
+    pub execution_payload: ExecutionPayload<P>,
+    pub versioned_hashes: VersionedHashes<P>,
+    pub parent_beacon_block_root: H256,
+}
+
```

### B2. `types/src/electra/containers.rs`

```diff
     deneb::{
-        containers::{ExecutionPayload, ExecutionPayloadHeader},
+        containers::{ExecutionPayload, ExecutionPayloadHeader, VersionedHashes},
         primitives::KzgCommitment,
     },
```

```diff
 pub struct ExecutionRequests<P: Preset> {
     pub deposits: ContiguousList<DepositRequest, P::MaxDepositRequestsPerPayload>,
     pub withdrawals: ContiguousList<WithdrawalRequest, P::MaxWithdrawalRequestsPerPayload>,
     pub consolidations: ContiguousList<ConsolidationRequest, P::MaxConsolidationRequestsPerPayload>,
 }
 
+/// Mirrors `NewPayloadRequest` from `electra/beacon-chain.md`.
+#[derive(Clone, PartialEq, Eq, Default, Debug, Deserialize, Serialize, Ssz)]
+#[serde(bound = "", deny_unknown_fields)]
+pub struct NewPayloadRequest<P: Preset> {
+    pub execution_payload: ExecutionPayload<P>,
+    pub versioned_hashes: VersionedHashes<P>,
+    pub parent_beacon_block_root: H256,
+    pub execution_requests: ExecutionRequests<P>,
+}
+
```

### B3. `types/src/bellatrix/containers.rs`

```diff
 pub struct ExecutionPayload<P: Preset> {
     ...
     pub transactions: Arc<ContiguousList<Transaction<P>, P::MaxTransactionsPerPayload>>,
 }
 
+/// Mirrors `NewPayloadRequest` from `bellatrix/beacon-chain.md`.
+#[derive(Clone, PartialEq, Eq, Default, Debug, Deserialize, Serialize, Ssz)]
+#[serde(bound = "", deny_unknown_fields)]
+pub struct NewPayloadRequest<P: Preset> {
+    pub execution_payload: ExecutionPayload<P>,
+}
+
```

### B4. `types/src/capella/containers.rs`

```diff
 pub struct ExecutionPayload<P: Preset> {
     ...
     pub withdrawals: ContiguousList<Withdrawal, P::MaxWithdrawalsPerPayload>,
 }
 
+/// Mirrors `NewPayloadRequest` from `capella/beacon-chain.md`.
+#[derive(Clone, PartialEq, Eq, Default, Debug, Deserialize, Serialize, Ssz)]
+#[serde(bound = "", deny_unknown_fields)]
+pub struct NewPayloadRequest<P: Preset> {
+    pub execution_payload: ExecutionPayload<P>,
+}
+
```

### B5. ssz_static wiring — `types/src/{bellatrix,capella,deneb,electra,gloas}/spec_tests.rs`

```diff
+tests_for_type! {
+    NewPayloadRequest<_>,
+    "consensus-spec-tests/tests/mainnet/<fork>/ssz_static/NewPayloadRequest/*/*",
+    "consensus-spec-tests/tests/minimal/<fork>/ssz_static/NewPayloadRequest/*/*",
+}
```

`tested_types` already does `containers::*`, so the name resolves. These globs match nothing until a `consensus-spec-tests` release built from post-#5619 includes the vectors; the pinned release (`v1.7.0-alpha.14`) predates it.

## Lodestar precedent — #10051

- `packages/types/src/{bellatrix,capella,deneb,electra,gloas}/sszTypes.ts`: `NewPayloadRequest` per fork; `VersionedHashes` in deneb; gloas is `ProgressiveContainerType(..., activeFields(4))`.
- `packages/types/src/types.ts`: `NewPayloadRequest` added to `TypesByFork` for bellatrix, capella, deneb, electra, fulu, gloas, heze.
- `specrefs/dataclasses.yml`: entries for bellatrix/deneb/electra.
- Verified against 896 vectors generated from spec commit `acd7a7f`, bellatrix through heze, minimal + mainnet.
- Nothing functional consumes it — only `packages/types` and `specrefs`; the engine still calls `notifyNewPayload(args)`. It is a type/conformance addition.
- Forcing function: Lodestar's `ssz_static` runner walks every vector on disk and asserts a type exists per `(fork, typeName)` (`packages/beacon-node/test/spec/presets/ssz_static.test.ts`). New vectors would fail without the types. Grandine's runner is an explicit `tests_for_type!` whitelist, so it will not fail — it just skips.

## Notes

- Part A removes imports used only by the struct (`ContiguousList`, `VersionedHash`, `ExecutionPayload`, `ExecutionRequests`, `Preset`); clippy catches a miss.
- Part A roots and all vectors are unchanged (same fields, order, `active = [1; 4]`).
- Part B adds no roots until wired to vectors; without B5 they are unused `pub` types.
- `combined.rs:1613` TODO (`ExecutionPayloadParams` vs `NewPayloadRequest`) is comment-only; Part B makes `ExecutionPayloadParams` overlap the spec's per-fork type — decide whether to keep both or converge.
- Sibling branches with `proof_engine/` need the same rename (`SszNewPayloadRequest` → `NewPayloadRequest`, path `eip8025` → `gloas`) when they land; not on this branch.
