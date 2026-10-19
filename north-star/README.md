# North star — a working Grandine optional-proof branch

**Status:** planning backlog, not an approved implementation design or delivery commitment.
**Created:** 2026-10-14. Baseline observations come from current Grandine PR1 / #17 at `94a265c872ed2ec47ab35767f97726c94d6a69ef` and local Prysm `optional-proofs` at `e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6`. Recheck revisions before implementation.

## The outcome we want

> Grandine imports a block and its accepted payload, receives a signed execution proof through its actual libp2p network, authenticates the envelope, reconstructs the expected statement, admits the attempt once, verifies a genuine proof with a real backend, stores the verified bare envelope, and reports the correct gossip outcome.

This is a **verifier-side optional-proof prototype**. Proof generation is external. It does not change payload validity, fork-choice weight/head or beacon-state transitions.

## Read this folder

- **[TODO.md](TODO.md):** full remaining implementation and evidence checklist, with completion criteria.
- **[MILESTONES.md](MILESTONES.md):** dependency order, demonstrable milestones and the conditional one-week target.

## System map

```text
Existing block/payload import ──> trusted block/state/accepted-payload context
                                                   │
External proof publisher                           │
        │                                          │
        ▼                                          │
eth2_libp2p: topic, subscription, bounded decode     │
        │                                          │
        ▼                                          │
p2p: signed envelope + GossipId                     │
        │                                          │
        ▼                                          ▼
fork_choice_control: background preparation using fork_choice_store helpers
        │
        ▼
mutator + Store: live atomic admission
        │
        ▼
background worker + proof_engine: real verification
        │
        ▼
mutator + Store: applicable completion, Seen, first-valid storage, pruning
        │
        └──> p2p / gossipsub: ACCEPT, REJECT or IGNORE
```

The normal `execution_engine/` boundary talks to the execution client. The separate `proof_engine/` boundary verifies cryptographic proofs. We are not implementing a new execution client.

## What already exists versus what remains

| Area | Observed baseline |
| --- | --- |
| Containers, limits and signing | Present; inherited from merged PR18 and earlier work |
| Authentication and payload-binding helpers | Present in current draft PR1 / #17; helper alignment is implemented, not evidence of PR approval/merge |
| Seen/proof map shapes | Present, but legal transitions and proof-specific pruning remain to implement |
| Controller/task/mutator skeleton | Present; proof task still returns stub IGNORE and spawn path is unwired |
| Real proof backend | Absent from the reviewed Grandine boundary; mock/null implementations only |
| Proof gossip | No execution-proof topic/pubsub variant in the reviewed `eth2_libp2p` source |
| Independent selected-target binding evidence | Outstanding; separate vector follow-up is planned |

The selected baseline remains master authentication/gossip order plus PR18’s deliberate SSZ extensions: #5643 progressive versioned hashes, bounded to 4,096 elements, and only #5642 progressive-list limit semantics. This is not pure-master SSZ interoperability. Prysm’s plain outer request container and encoded-public-values convention are not automatically compatible with it.

## Explicitly outside this north star

- Recursive proofs, proof aggregation and ancestor-proof validity propagation.
- Proof generation inside Grandine.
- Proof-count thresholds or proof-only payload import/validity gating.
- Changes to fork-choice weight/head or beacon-state transitions from proof outcomes.
- Proof-specific Req/Resp, historical serving and durable proof/Seen/pending storage.
- Requiring both HTTP and gossip ingress; **gossip is the first prototype ingress**.
- A generalized token/recovery framework or production-scale hardening in the first prototype.

Correctness, bounded network admission and honest failure classification are still prototype requirements, not optional production extras.

## Evidence and related planning

- [PR2 decision sheet](../execution-proof-pr2-master-update/DECISIONS.md).
- [PR2 draft design](../execution-proof-pr2-master-update/0004-pr2-master-design-draft.md).
- [PR1/master evidence and action map](../execution-proof-pr2-master-update/README.md).
- [Prysm comparison and limitations](../execution-proof-pr2-master-update/PRYSM-PR2-EVIDENCE.md).
- [Independent SSZ-vector follow-up](../eip8025-ssz-vectors/README.md).

Unchecked backlog items are potential remaining work, not proof that a chosen implementation must use every mechanism named. Record evidence when completing an item; do not check it solely because a helper, interface or test name exists.
