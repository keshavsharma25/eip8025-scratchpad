# Implementation checklist — working optional-proof branch

**Status:** proposed backlog. No new implementation is claimed. Decision 1 was approved on 2026-10-14 as A — permanent Seen; all pending/token/release alternatives below are superseded for the current prototype. Resolve remaining choices in the operative design before coding; check implementation items only with implementation/test evidence.

## A. Lock the receiving-side contract

- [x] Record PR2 decision 1: permanent Seen for prototype simplicity, no local-fault retry or pending/token machinery.
- [x] Record PR2 decision 2’s scope reduction: retain spec bool and existing task opt-out guard; pre-engine local faults belong to task orchestration, not a new PR2 recovery/API framework.
- [x] Record PR2 decision 3: require relevant live-store applicability at admission/completion; no pointer/global-generation scheme selected. Exact predicate/provenance still need engineering evidence.
- [ ] Record the three remaining [PR2 decisions](../execution-proof-pr2-master-update/DECISIONS.md): retention, distinct concurrent attempts and PR2 scope/vector sequencing.
- [ ] Establish exact trusted state/payload provenance and the cheap live context identity from Grandine code.
- [ ] Establish the exact retained-block predicate, finalized-anchor/orphan treatment and local retirement-check placement.
- [ ] If pending is selected, specify paired ownership, cessation evidence and release rules; never release on timeout alone.
- [ ] Reconcile the draft/design and scope table with the decisions; re-pin and review before treating it as an implementation contract.

**Complete when:** the state-machine invariants are unambiguous and no pending policy is selected merely from a filename or earlier recommendation.

## B. Implement the preparation and store state machine — PR2

Relevant surface: `grandine/fork_choice_store/src/store.rs`, `error.rs`, `misc.rs`.

- [ ] Implement exact master preparation precedence: early empty/type rejection, block-known, verified-slot, bare root, Seen-root/prover, payload availability, authentication and binding.
- [ ] Materialize the target block’s trusted post-state off the mutator; distinguish local loading/cache failure from peer invalidity.
- [ ] Reuse current PR1 auth/binding; require the accepted marker and corresponding trusted payload context, not peer-chosen public input.
- [ ] Introduce narrow prepared/admitted/completion-input provenance and typed outcomes that cannot bypass verification through arbitrary apply calls.
- [ ] Implement live atomic admission: check verified storage, both duplicate keys, context and retention; insert/reserve both keys or neither.
- [ ] Implement the selected permanent-Seen or owned-pending lifecycle.
- [ ] Implement completion applicability checks and first-valid bare-envelope storage.
- [ ] Keep applicable definitive-invalid attempts Seen; never store them as verified proofs.
- [ ] Prune all proof record types consistently; prevent late completion or historical links from resurrecting retired records.
- [ ] Add store tests for ordering, authentication/no-consumption, paired exclusion/no partial writes, replay, distinct attempts, first-valid storage, context and pruning; add token/release cases if pending.

**Complete when:** preparation/store transitions pass scoped checks. This does not establish a running end-to-end pipeline.

## C. Execute the pipeline, rather than the stub

Relevant surface: `grandine/fork_choice_control/src/{controller,tasks,messages,mutator,thread_pool}.rs`.

- [ ] Provide a controller submission entry point carrying a signed envelope and its origin.
- [ ] Replace stub IGNORE with background preparation, live mutator admission, admitted engine work and mutator completion.
- [ ] Preserve non-blocking handoffs; do not occupy a worker waiting for a mutator admission reply.
- [ ] Publish snapshots after bookkeeping mutations so later preparation sees current records.
- [ ] Preserve exact admitted input/context through completion; no rebind against a replacement payload.
- [ ] Transfer origin ownership and produce one normal terminal result.
- [ ] Define scheduling/non-start/completion behavior and supported stopped-local-failure reporting; keep uncertain running occupancy if pending.
- [ ] Preserve null/disabled bypass before preparation/admission.
- [ ] Add controlled interleaving tests asserting actual engine calls, terminal outcomes and late-result behavior.

**Complete when:** internal submission traverses the full pipeline with a controllable mock verifier; same-key races do not cause duplicate admitted calls under the selected lifecycle.

## D. Integrate a real verifier

Relevant surface: `grandine/proof_engine/` and its startup/caller integration.

- [ ] Select an existing backend and one initial proof type/guest version; do not implement a zkVM or prover.
- [ ] Implement the adapter and proof decoding/cryptographic verification.
- [ ] Configure proof-type → guest/program verification-key mapping.
- [ ] Check the proof against Grandine’s locally reconstructed statement, not arbitrary proof-carried public values.
- [ ] Specify valid/invalid/local-failure behavior. Extend the current boolean boundary if the backend requires operational errors; do not map unavailability to false.
- [ ] Add finite verifier concurrency and define its waiting/overload behavior.
- [ ] Document cancellation limits; do not claim that a context deadline interrupts a native call without evidence.
- [ ] Test genuine valid proof, corrupted proof, wrong statement, missing configuration and operational failure.

**Complete when:** the adapter verifies a genuine compatible fixture independently of network plumbing and preserves honest error classification.

## E. Establish compatibility and independent fixtures

- [ ] Pin network/fork/config, selected SSZ revisions, proof type, guest version, verification key and proof-byte format.
- [ ] Verify how the backend/guest commits public input and reconcile it with our intended boundary. Any change to the selected contract needs an explicit design decision.
- [ ] Obtain independently sourced payload-derived request/public-input roots for our selected target.
- [ ] Generate/check encoding, decoding and roots through the separate vector workstream.
- [ ] Obtain a genuine proof plus all context needed to reconstruct its statement.
- [ ] Prove that external producer and Grandine derive the same statement; do not use Prysm’s different outer-container root as our oracle.
- [ ] Add negative fixtures for altered proof and mismatched payload/public input.

**Complete when:** real interoperability is demonstrated for the explicitly pinned target. Progressive lists alone or Grandine-only expected roots are insufficient.

## F. Add execution-proof gossip to eth2_libp2p

Relevant surface: `grandine/eth2_libp2p/src/types/{topics,pubsub}.rs`, service/discovery/scoring integration.

- [ ] Add the selected target’s proof topic(s) and signed-envelope pubsub message variant. The current baseline has global `execution_proof`; recheck open consensus-specs #5726 (type-mapped subnets) before network integration and explicitly align the target if adopted. Topic partitioning is not per-key attempt exclusion.
- [ ] Implement topic-name parsing/display, encoding, decoding and all required enum match integration.
- [ ] Subscribe under the correct fork digest and feature enablement; handle existing fork/subscription lifecycle consistently.
- [ ] Add the spec’s `eproof` ENR awareness field consistently with actual enabled support.
- [ ] Ensure controlled proof-aware peers connect and form the demonstration mesh. Preferential discovery can follow; document any explicit-peer restriction.
- [ ] Integrate applicable subscription filters/topic scoring/metrics without inventing measured rates or inappropriate rejection penalties.
- [ ] Test topic round-trip, SSZ/Snappy round-trip, supported decode behavior, disabled subscription and malformed messages.

**Complete when:** the actual libp2p service receives a correctly encoded signed execution-proof envelope from a peer. No proof-specific Req/Resp is required.

## G. Connect networking to the proof service and back

Relevant surface: `grandine/p2p/src/network.rs`, controller entry point and existing validation-result channels.

- [ ] Dispatch decoded proof + source/message identity into the controller as a gossip origin.
- [ ] Route ACCEPT/REJECT/IGNORE back through existing gossipsub validation reporting.
- [ ] Ensure definitive invalidity and local unavailable-context/backend/capacity failures have distinct peer-scoring behavior.
- [ ] Exercise successful propagation after acceptance through the normal gossip mechanism.
- [ ] Keep generic error handling from turning every local error into peer rejection.
- [ ] Add network/service integration tests for valid, invalid, ignored and duplicate attempts.

**Complete when:** receiving bytes leads to the correct service transition and network validation outcome, not merely a logged decode.

## H. Bound resources before enabling live ingress

- [ ] Enforce the 4,194,449-byte signed-envelope cap at the appropriate transport boundary, including before decompression allocation where applicable.
- [ ] Verify compressed-frame and decoded-size handling; keep bounded ProofData construction/decode tests.
- [ ] Bound admitted/in-flight proof work, queued bytes/work and verifier concurrency with documented prototype values.
- [ ] Specify local overload outcomes without fabricating a spec-invalid proof verdict.
- [ ] Test oversize/compression cases, saturation and continued progress of ordinary node work.
- [ ] Document shared-worker limitations: low priority is neither isolation nor cancellation.

**Complete when:** live proof input is finite and overload/failure handling is observable and non-punitive where appropriate. Production-scale tuning is deferred, not basic admission safety.

## I. Configure startup, disabled mode and shutdown

- [ ] Add explicit enablement and backend/key configuration to the existing node configuration path.
- [ ] Construct and inject the verifier; avoid production startup with a success-mock by accident.
- [ ] Align backend readiness, gossip subscription and advertised proof awareness.
- [ ] Fail startup clearly on unusable enabled configuration; define unsupported-platform behavior.
- [ ] Test disabled mode, enabled mode and configuration errors.
- [ ] Define shutdown behavior consistent with running work/attempt lifecycle; no durable restart deduplication guarantee is required.

**Complete when:** launching a configured node activates the feature without code edits or manually constructed test objects.

## J. Make the backend build reproducible

- [ ] Pin/provision native libraries and other backend artifacts through project-appropriate build mechanisms.
- [ ] Document supported platforms, build features, key provenance and required artifacts.
- [ ] Add supported-backend compilation/test coverage to CI; ensure intentionally disabled builds behave as documented.
- [ ] Run formatting, scoped tests/checks/Clippy and required broader CI before sign-off; distinguish scoped success from full-workspace validation.
- [ ] Document limitations rather than silently treating absent dependencies as proof invalidity.

**Complete when:** another developer can build and run the branch in a documented environment. No global tool installation or remote-code execution is authorized by this checklist.

## K. Add minimal diagnostics

- [ ] Distinguish receipt, preparation decline, authentication failure, admission, engine verdict, local fault, storage/context discard and pruning.
- [ ] Add basic counters/timing where the existing metrics framework supports them; no new observability framework is required.
- [ ] Avoid proof bytes/signatures in logs and high-cardinality validator/block labels in metrics.
- [ ] Provide a test assertion or development inspection mechanism proving the verified bare envelope was stored.
- [ ] Document how to tell peering failures from context, signing, binding and backend failures.

**Complete when:** failed demonstrations can be diagnosed and successful verification/storage is evidenced, not inferred from “message received”.

## L. Deliver a reproducible end-to-end demonstration

- [ ] Provision a compatible controlled network or fixture-backed setup with a block and accepted payload known to Grandine.
- [ ] Provide a compatible external publisher/producer and eligible validator signatures; no proof generation inside Grandine.
- [ ] Match fork digest, signing domain, state context and statement target.
- [ ] Demonstrate peer publish → Grandine receive → authentication → admission → real verification → bare-proof storage → gossip outcome.
- [ ] Demonstrate bad signature, authenticated-invalid proof, wrong statement and duplicate/concurrent suppression.
- [ ] Demonstrate local failure/disabled/capacity handling without unjustified peer penalty.
- [ ] Demonstrate finalization + late completion without resurrection and oversized/malformed network input rejection.
- [ ] Write setup/run commands, fixture provenance, expected observations and supported limitations.

**Complete when:** someone else can reproduce the real-proof network demonstration. A mock-backed demo is a separate milestone, never described as cryptographic verification.

## Deferred / excluded — do not add silently

- Recursive proofs, aggregation or ancestor-proof validity propagation.
- Prover implementation or generation APIs.
- Proof-driven fork-choice/head/state/payload-validity changes and proof-count thresholds.
- Proof-specific Req/Resp, historical serving, durable proof/Seen/pending storage and restart recovery.
- HTTP proof submission in addition to the first gossip path.
- Generalized token/lease infrastructure, broad production tuning or all proof systems before the first one works.

This backlog does not assign every item to PR2 and does not authorize implementation, commits or GitHub writes. Use the existing project harness and preserve minimal, explicitly scoped diffs.
