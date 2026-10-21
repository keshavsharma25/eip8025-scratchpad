# Execution-proof PR stack

A concise handoff for Grandine’s EIP-8025 verifier work: what each PR owns, the selected design, and what remains. These pages stand alone; personal scratchpad notes are not required.

## Read in this order

1. [Decisions](DECISIONS.md) — selected policies, rationale and alternatives.
2. [PR1: preparation](PR1.md) — authentication, trusted context and protected input.
3. [PR2: Store lifecycle](PR2.md) — admission, completion and pruning.
4. [PR3: integration](PR3.md) — workers, messages, engine calls and signalling.

**Numbering:** PR1 is repository PR #17. PR2 and PR3 are logical stack positions; no repository PR numbers are assigned here. Repository PR #18 is the inherited data-model change, not lifecycle PR2.

```text
worker:  PR1 prepare/authenticate/bind
mutator: PR2 live admission + paired Seen insertion
worker:  PR3 run engine on exact admitted input
mutator: PR2 complete/store; PR3 publish snapshot + signal origin
         PR2 prune all proof bookkeeping when targets retire
```

## Status snapshot

Local inspection on 2026-10-20: branch `feature/proof-envelope-auth-binding`, HEAD `a5f27d52070d321bc10ca754dd77c542fd276030`. No changes appeared in the scoped Store/control/engine/manifests status check.

| Stage | What is established | What is not established |
| --- | --- | --- |
| PR1 | Preparation implementation exists, including protected input and exact-target context loading. | No fresh build/test or merge-readiness claim in this handoff. |
| PR2 | Recorded CP1 contract accepted; CP2 cheap live-applicability gate remains unresolved. | No implemented lifecycle is claimed. Proof pruning is still a TODO. |
| PR3 | Existing plumbing and null-engine guard exist. Engine boundary is now `ProofEngine` with a consuming input and boolean verdict. | Enabled proof task still returns stub IGNORE; end-to-end verification is not wired. |

The checkpoint statuses above are the latest recorded work state, not newly executed acceptance checks. No fresh GitHub, merge-state, Cargo, lint or interoperability verification was performed for this documentation.

## Scope limits

No proof generation, new live ingress, or proof-driven changes to payload validity, head/weight or beacon state. Independent binding vectors and real-backend interoperability are a separate workstream. Finite resource limits and transport size enforcement are required before live exposure.

## Relevance of the older decision sheet

Its six policy choices remain the basis of this stack: permanent Seen, boolean verification, live applicability, retained-block pruning, distinct-attempt overlap and bounded PR ownership. Its October 15 baseline, old trait names and statements that PR1 preparation is only planned are stale. Use the decisions here for the concise contract; do not reuse the old sheet as a progress or readiness report.
