# Milestones — from helpers to a networked real-proof prototype

**Status:** proposed sequencing, not a promised schedule. See [TODO.md](TODO.md) for acceptance work and [README.md](README.md) for scope.

## M0 — settle PR2’s contract

**Work:** TODO A; backend/fixture feasibility investigation from D/E can proceed independently.

**Demonstration:** recorded lifecycle/context/retention/concurrency choices and code-supported invariants. No runtime prototype claim.

**Gate:** finish the current [PR2 decision discussion](../execution-proof-pr2-master-update/DECISIONS.md) one choice at a time. Do not use the north star to bypass unsettled store semantics.

## M1 — tested preparation/store transitions

**Work:** TODO B.

**Demonstration:** unit tests prove ordered preparation, atomic paired-key admission, completion/storage, selected failure policy and pruning.

**Gate:** no unauthenticated key consumption, partial claim insertion, invalid-proof storage or late-result resurrection. This is the PR2 contract milestone, not an end-to-end service.

## M2 — running internal mock-backed pipeline

**Work:** TODO C and relevant diagnostic/configuration seams.

**Demonstration:** internal submission traverses preparation → admission → engine → completion/storage. Controlled interleavings assert actual mock-engine calls.

**Gate:** no duplicate admitted same-key verification under the selected retained-key lifecycle, correct terminal outcomes, no expensive mutator verification.

**Label:** service-mechanics prototype; no cryptographic proof validity claim.

## M3 — mock-backed gossip prototype

**Work:** TODO F/G/H and enablement from I; extend C/K tests and diagnostics.

**Demonstration:** explicitly connected peers publish signed envelopes through the real `eth2_libp2p` path; Grandine runs the service and reports gossip outcomes.

**Gate:** bounded ingress/work, correct fork/topic/signing context, disabled behavior, malformed/oversized handling and honest peer scoring.

**Label:** network/service prototype; still not real execution-proof verification. Explicit peer connections are acceptable for the controlled demo; document discovery limitations and implement awareness consistently.

## M4 — standalone real verifier + compatible statement

**Work:** TODO D/E and required backend build/configuration pieces from I/J.

**Demonstration:** a genuine proof verifies against the exact expected statement; altered bytes/wrong statement fail, and operational errors remain distinct.

**Gate:** independent statement/fixture provenance, agreed guest/key/public-input convention. Prysm’s deliberately different root conventions are not our oracle.

**Parallelism:** feasibility can be established before M1–M3 finish. Integrating an available backend does not depend on network-topic implementation; neither workstream removes the other’s acceptance gates.

## M5 — working optional-proof branch

**Work:** combine M3 + M4; complete startup/build/diagnostics and end-to-end acceptance work in I/J/K/L.

**Demonstration:** genuine signed proof from a peer → Grandine’s actual gossip/service path → real verification → stored bare envelope → correct network outcome, with reproducible commands and negative cases.

**Gate:** distinguish prototype limitations from demonstrated properties. No recursive proofs, proof generation or proof-driven payload-validity policy is required.

## Dependency sketch

```text
M0 -> M1 -> M2 -> M3 --+
                      +-> M5
backend/fixture -> M4 -+

Configuration, finite resource admission, build support and diagnostics
must accompany the stages that depend on them, not be postponed until after exposure.
```

## One-week framing

**Conditional target discussed:** M3, a bounded controlled-peer mock-backed gossip prototype. Even this is ambitious; it depends on timely PR2 decisions and integration/testing effort. It is a planning target, not a commitment.

**Stretch:** M5 with one real proof type if backend/fixture compatibility is demonstrated early. Do not assume one week covers compatibility debugging plus the service and networking work.

**Early go/no-go check:** can an existing real verifier accept a genuine fixture against the exact statement Grandine intends to reconstruct? If not, explicitly retain M3 as the milestone and track D/E separately; do not claim M5 because the network accepts mock proofs.

**Scope control:** gossip first; one initial real proof type; controlled peers; no extra HTTP ingress, recursive proof features, proof generation, historical serving or durable recovery. Do not weaken ownership/freshness/retirement invariants to meet a date.

## How to update progress

For each completed milestone, record:

- Exact Grandine/backend/spec/guest/key/fixture revisions.
- Implemented files and scope.
- Commands/tests run and passed/failed/skipped results.
- Demonstration evidence and known limitations.

Do not translate a milestone into “percent complete”: verifier compatibility and concurrency integration can dominate remaining effort even when most files or checkboxes exist.
