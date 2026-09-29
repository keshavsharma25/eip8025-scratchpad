# PR2 — stale-snapshot sequence diagrams

**Status:** teaching aid for the [interview decision log](decisions.md), not an implemented flow. Based on design 0004's pinned consensus-specs `6946894b02f6e95bf5b1daf4ce39a3c3244a1e41`. These diagrams cover meaningful **time windows**, not every fork-choice mutation. A newer snapshot **alone** does not invalidate a proof: the relevant block, post-state, accepted payload/binding, Seen keys, stored proof, and claim ownership determine the outcome. The exact Rust context identity, claim lifecycle, and recovery APIs remain open. Arrows from the mutator to an origin below mean **result signalling**, not a direct mutator-to-peer network call. `R` is the bare-message root; `K` is `(block root, proof type, validator index)`.

## 1. Stage 1 and a stale snapshot **before claim grant**

```mermaid
sequenceDiagram
    autonumber
    participant P as Gossip origin / result sink
    participant S as Snapshot S0 (read-only)
    participant W1 as Stage-1 worker
    participant M as Mutator + live Store
    participant W2 as Stage-2 engine worker

    P->>W1: Deliver signed proof X
    W1->>S: Read a coherent S0
    S-->>W1: Block, post-state, accepted payload, Seen and stored data
    W1->>W1: Ordered prechecks, authenticate envelope, bind engine input
    Note over W1,M: Only a stage-1 success sends a claim request. No snapshot grants engine access.
    W1->>M: ClaimRequest(R, K, bound input, context, origin)
    Note over M: Recheck LIVE state in pinned check precedence before granting.
    alt R is Seen or pending
        M-->>P: IGNORE, no engine call
    else Block is unknown
        M-->>P: IGNORE, no engine call
    else Known block lacks valid post-state
        M-->>P: REJECT (D2), no engine call
    else Accepted payload is unavailable
        M-->>P: IGNORE (D1), no engine call
    else Verified proof for block and type is stored
        M-->>P: IGNORE, no engine call
    else K is Seen or pending
        M-->>P: IGNORE, no engine call
    else Post-state or accepted payload differs from bound context
        M-->>P: IGNORE, no engine call
    else Context matches and both keys are free
        M->>M: Atomically reserve R and K with fresh token t
        M->>W2: Schedule bound input with token t
        W2->>W2: Verify bound input
        W2-->>M: Finish(t, verdict)
        M->>M: Recheck both-key ownership and relevant live context
        alt Token or binding invalid
            M->>M: Discard stale result, do not mark Seen
            M-->>P: IGNORE
        else Engine verdict is true
            M->>M: Clear pending, mark both Seen, store if slot still empty
            M-->>P: ACCEPT
        else Engine verdict is definitively false
            M->>M: Clear pending, mark both Seen, do not store proof
            M-->>P: REJECT
        end
    end
```

Stage 1 may itself IGNORE/REJECT before sending any claim request. The claim-time live checks preserve the pinned early-check precedence: a known block without a valid post-state is REJECT even if its payload is also missing. Authentication is **not** repeated on the mutator; the still-open context-identity design must enforce the binding cheaply.

**Old-snapshot race:** A and B both read S0 with empty Seen and authenticate. A's claim reserves `(R,K)` first. When B's claim is processed, the live pending root or prover key is present, so B gets IGNORE without an engine call. Both may already have paid for hashing/BLS. Neither worker mutates S0 or live Seen.

## 2. Relevant context changes **after grant** (before start, during work, or while Finish waits)

```mermaid
sequenceDiagram
    autonumber
    participant P as Gossip origin / result sink
    participant E as Store event
    participant M as Mutator + live Store
    participant W2 as Stage-2 engine worker
    participant Q as Finish queue

    M->>M: Grant X against context C0, reserve R and K with token t
    M->>W2: Schedule bound input(C0), token t
    alt Context lost before queued task starts
        E->>M: Prune block or lose accepted payload
        M->>M: Invalidate claim t
        W2->>W2: Queued work may still start and verify C0
        W2-->>Q: Enqueue Finish(t, verdict)
    else Context changes while engine runs
        W2->>W2: Begin verifying C0
        E->>M: Relevant block, state or accepted payload changes
        M->>M: Invalidate claim t when change is observed
        W2->>W2: Complete verification of C0
        W2-->>Q: Enqueue Finish(t, verdict)
    else Context changes after engine returns
        W2->>W2: Verify C0 and return
        W2-->>Q: Enqueue Finish(t, verdict)
        E->>M: Relevant context changes before Finish is handled
        M->>M: Invalidate claim t when change is observed
    end
    Q-->>M: Deliver Finish(t, verdict)
    M->>M: Check ownership of BOTH keys and live binding against C0
    M-->>P: IGNORE, no Seen or verified-proof write
```

These branches illustrate a result that *does* arrive; a canceled queued task might never call the engine or send Finish. Cleanup for non-completion is still open. Even if invalidation is not performed eagerly on a store event, the finish-time live check must prevent stale writes. The engine receives only the input prepared from S0, not S0 itself or write authority. A `true` result cannot resurrect pruned data. An unrelated snapshot publication with **unchanged relevant context and valid token** is not grounds for IGNORE. Another validator storing a valid proof is also not, by itself, a binding change for an already admitted claim (diagram 3).

## 3. A stored proof and an unrelated snapshot update do **not** cancel an admitted claim

```mermaid
sequenceDiagram
    autonumber
    participant PA as Origin of proof A
    participant PB as Origin of proof B
    participant M as Mutator + live Store
    participant EA as Engine task A
    participant EB as Engine task B

    Note over PA,PB: A and B have different bare messages and different prover keys, but the same block and type.
    PA->>M: Authenticated ClaimRequest(A)
    M->>M: Reserve A with token a, no verified proof stored
    M->>EA: Schedule A
    PB->>M: Authenticated ClaimRequest(B)
    M->>M: Reserve B with token b, no verified proof stored
    M->>EB: Schedule B
    M->>M: Publish an unrelated snapshot, binding unchanged
    EA-->>M: Finish(a, true)
    M->>M: Check a and context, commit A Seen and store valid A
    M-->>PA: ACCEPT A
    Note over M,EB: Storing A does not invalidate B's already granted token or binding.
    EB-->>M: Finish(b, false)
    M->>M: Check b and context, commit B Seen, store no B
    M-->>PB: REJECT B (definitively invalid)
    Note over PB,M: A new proof C for the same block and type is IGNORE before engine entry.
```

If B's engine returned `true`, B would be ACCEPTED as an already admitted proof, but A's verified proof would **not be overwritten**. If A returned `false`, A would be REJECTED with **no verified proof** stored, so B could still succeed. These are the agreed Q13/Q14 local interleaving rules, not a concurrent ordering supplied by the sequential spec. If A and B have the **same bare message**, B conflicts on the pending root even though the validator indices differ: B gets IGNORE at claim and has no engine task. A new attempt arriving *after* A was stored gets IGNORE; it is not the already admitted B.

## 4. Delayed results versus a replaced claim (token fencing)

```mermaid
sequenceDiagram
    autonumber
    participant PO as Old result sink
    participant PN as New result sink
    participant E as Store event
    participant M as Mutator + live Store
    participant Old as Old engine task
    participant Q as Finish queue
    participant New as New engine task

    M->>Old: Grant R and K with token t1, verify bound C0
    Old->>Old: Complete verification and stop
    Old-->>Q: Enqueue Finish(t1, true), delivery delayed
    E->>M: Accepted payload temporarily unavailable
    M->>M: Invalidate t1, no Seen or proof written
    E->>M: Context becomes available again
    M->>M: Confirm old worker stopped, recheck live admission
    M->>New: Grant same free keys with fresh token t2, start new task
    Q-->>M: Deliver delayed Finish(t1, true)
    M->>M: t1 no longer owns R and K, leave t2 intact
    M-->>PO: IGNORE old result, no stale write
    New-->>M: Finish(t2, true)
    M->>M: Check t2 owns both keys and context matches, mark Seen and store proof
    M-->>PN: ACCEPT new result
```

This is an **illustrative** delayed-message race, not a specified recovery API. The old task has **stopped** before the same keys are granted again: a timer alone must not release t1 while it could still execute. A pruned block's keys cannot be regranted merely to demonstrate fencing; pruning instead makes any late result IGNORE without resurrecting its records. Exactly how a stopped task is confirmed and a delayed result is abandoned is still open. Token fencing prevents stale commits, **not** overlapping engine work if claims are released prematurely.

**Resource caveat:** same-key engine calls cannot overlap in the ordinary granted-claim path. Many *different* authenticated keys and repeated pre-claim hashing/BLS can still consume resources. Admission, queue, and concurrency bounds are required before real gossip ingress is enabled; this design is not a DDoS guarantee.

## Decision checkpoints (read along the diagrams)

| When live state changes | Which decision uses live state? | Proposed outcome |
| --- | --- | --- |
| Before stage-1 worker reads | Worker reads the snapshot supplied | Recheck at claim; snapshot alone never grants engine access |
| Between snapshot read and ClaimRequest handling | Mutator checks live R/Seen/pending, block/state/payload, stored proof, K/Seen/pending, then binding compatibility in pinned early-check precedence | Duplicate or missing block/payload: IGNORE; known block with missing post-state: REJECT; changed binding: IGNORE; no engine |
| After claim grant, before stage-2 starts | Mutator may invalidate claim; queued worker may still run | On finish: IGNORE if token/context invalid; no stale writes; pre-start cancellation not yet specified |
| During engine work | Mutator handles store changes while worker uses bound C0 | Finish-time live check; changed relevant context: IGNORE, no Seen/proof |
| After engine returns, before Finish is handled | Queued result carries token, not write authority | Mutator rechecks; invalidated token or changed binding: IGNORE |
| Only unrelated snapshot update | Binding and token remain valid | Honor the engine result; do **not** discard solely because a new snapshot was published |
| Different validator's valid proof stored while B's earlier claim runs | B's token/context remain valid | B gets its own engine verdict and Seen; store first valid proof only (Q14) |
| Delayed old result after a replacement grant | Mutator checks ownership of both pending keys | Old token: IGNORE without touching new claim; do not grant replacement while old engine may still run |

**Open implementation details:** exact context identity and material-change policy; how stopped workers and abandoned results are detected without timer-only release; handling enqueue failure/panic/shutdown; bounded pending and verifier work; PR2/PR3 division. See [decisions.md](decisions.md) for agreed status and [report.md](report.md) for the in-depth walkthrough.
