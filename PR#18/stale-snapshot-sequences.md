# PR2 — stale-snapshot sequence diagrams

**Status:** teaching aid for the [interview decision log](decisions.md), not an implemented flow. Based on design 0004's pinned consensus-specs `6946894b02f6e95bf5b1daf4ce39a3c3244a1e41`. These diagrams cover the meaningful **time windows**, not every possible fork-choice mutation. A changed snapshot **alone** does not invalidate a proof: the *relevant* block, post-state, accepted payload/binding, Seen keys, stored proof, and claim ownership determine the outcome. The exact Rust context identity and recovery APIs remain open.

## 1. The normal two-stage path and a stale snapshot **before claim grant**

```mermaid
sequenceDiagram
    autonumber
    participant P as Incoming proof / peer
    participant S as Snapshot S0 (read-only)
    participant W1 as Stage-1 worker
    participant M as Mutator + live Store
    participant W2 as Stage-2 engine worker

    P->>W1: Deliver signed proof X
    W1->>S: Read S0; ordered prechecks, authenticate, bind
    S-->>W1: Prepared X + bound input + context identity
    Note over W1,M: S0 can become old immediately; a worker cannot update live Seen.
    W1->>M: ClaimRequest(X, root R, prover key K, context)
    Note over M: Recheck LIVE context and Seen/stored/pending keys before any engine call.
    alt Live root/prover key is now Seen or pending, or valid (block,type) proof is stored
        M-->>P: IGNORE; do not start engine
    else Block missing, or payload missing / materially changed
        M-->>P: IGNORE; do not start engine
    else Known block has no valid post-state
        M-->>P: REJECT under design 0004 D2; no claim
    else Context still matches and both keys are free
        M->>M: Atomically claim R and K with fresh token t
        M->>W2: Schedule engine task(bound input, token t)
        W2->>W2: verify_execution_proof(input)
        W2-->>M: Finish(t, true or false)
        M->>M: Recheck token + relevant LIVE context
        M-->>P: ACCEPT if true, REJECT if false (if still applicable)
    end
```

**Example:** A and B both read S0 with empty Seen. A's ClaimRequest reaches the mutator first and reserves `(R,K)`. B's ClaimRequest still carries S0, but the *live* pending key now exists: B gets IGNORE and does not call the engine. Both may have paid for BLS authentication already. No snapshot update retroactively alters S0; the mutator's live check is what protects engine entry.

## 2. Context changes **after grant**, before or during the engine, or while the result is queued

```mermaid
sequenceDiagram
    autonumber
    participant P as Proof X / peer
    participant M as Mutator + live Store
    participant W2 as Stage-2 engine worker
    participant F as Finalization / other store event

    M->>M: Claim granted for X using context C0; reserve R and K with token t
    M->>W2: Schedule bound input(C0) + token t
    alt Relevant context changes before W2 starts
        F->>M: Prune block, lose accepted payload, or change bound context
        M->>M: Remove/invalidate claim as applicable
        Note over W2: A queued task may still run; no promise of zero wasted engine work.
    else Relevant context changes during verification
        W2->>W2: Verify input bound to C0
        F->>M: Prune block or change relevant context to C1
        M->>M: Invalidate/prune token t as applicable
    else Relevant context changes after engine finishes but before mutator handles Finish
        W2->>W2: Verify input bound to C0
        F->>M: Prune block or change relevant context to C1
    end
    W2-->>M: Finish(t, true or false)
    M->>M: Check token ownership AND live context matches C0
    M-->>P: IGNORE; do not write Seen or verified proof
    Note over M,P: Even an engine true result cannot resurrect a pruned block.
```

**Correction to the motivating example:** the engine does **not** receive a store snapshot or decide whether a result may update the store. It receives a proof input built from the stage-1 snapshot; its `bool` result goes back to the **mutator**. If relevant context changed while the result was in flight, the mutator signals IGNORE and discards it. An unrelated tick/snapshot publication with *unchanged relevant context* is **not** grounds to ignore. Snapshot freshness is checked at claim grant and result application, not continuously during a synchronous engine call.

## 3. Two other snapshot updates that must **not** be conflated with context loss

```mermaid
sequenceDiagram
    autonumber
    participant PA as Proof A / validator 1
    participant PB as Proof B / validator 2
    participant M as Mutator + live Store
    participant EA as Engine task A
    participant EB as Engine task B

    Note over PA,PB: A and B have different bare messages (different roots) and different prover keys.
    PA->>M: Authenticated ClaimRequest(A)
    M->>M: Reserve A, token a; no verified (block,type) proof yet
    M->>EA: Start A
    PB->>M: Authenticated ClaimRequest(B)
    M->>M: Reserve B, token b; no verified proof yet (agreed Q13)
    M->>EB: Start B
    EA-->>M: Finish(a, true)
    M->>M: A's context valid; mark A Seen; store valid A
    M-->>PA: ACCEPT A
    Note over M,EB: B's snapshot is older, but A's storage alone does not revoke B's granted claim.
    EB-->>M: Finish(b, false)
    M->>M: B's token and binding still valid; mark B Seen; store no B
    M-->>PB: REJECT B (agreed Q14, accurate peer scoring)
    Note over PB,M: A newly arriving proof C for the same (block,type) is IGNORE before engine.
```

If B's engine returned `true` instead, B would be ACCEPTED as an already admitted proof, but A's verified proof would **not be overwritten**. If A returned `false`, A would be REJECTED and **no verified proof** would be stored; B could still succeed. If A and B have the **same bare message**, B conflicts on the pending message root even though their validator indices differ: B gets IGNORE **at claim**, no engine B.

## 4. Old results versus a replaced claim (token fencing)

```mermaid
sequenceDiagram
    autonumber
    participant Old as Old engine task
    participant M as Mutator + live Store
    participant New as New engine task
    participant P as Peer

    M->>Old: Grant (R,K) with token t1
    Note over M: Finalization or confirmed task cessation removes t1; timeout alone is NOT release.
    M->>M: Later grant for same keys gets fresh token t2 if context/retention allows
    M->>New: Start task with t2
    Old-->>M: Late Finish(t1, true)
    M->>M: t1 no longer owns both keys; ignore old result
    M-->>P: IGNORE old result; no stale Seen or proof write
    New-->>M: Finish(t2, true or false)
    M->>M: Only t2 may commit if context is still valid
```

**Resource caveat:** same-key engine calls cannot overlap in the ordinary granted-claim path. A mere deadline must not grant t2 while t1 could still be executing; otherwise a token prevents stale writes but **does not prevent duplicate engine work**. Many *different* authenticated keys and repeated pre-claim hashing/BLS can still consume resources. Admission/queue/concurrency bounds are required before real gossip ingress is enabled; this design is not a DDoS guarantee.

## Decision checkpoints (read along the diagrams)

| When live state changes | Which decision uses live state? | Proposed outcome |
| --- | --- | --- |
| Before stage-1 worker reads | Worker reads whatever snapshot it is given | Recheck at claim; snapshot alone never grants engine access |
| Between snapshot read and ClaimRequest handling | Mutator checks live Seen, pending, stored proof, block/state/payload context | Duplicate or missing payload/block: IGNORE; known block with missing post-state: REJECT; no engine |
| After claim grant, before stage-2 starts | Mutator may invalidate claim; queued worker may still run | On finish: IGNORE if token/context invalid; no stale writes; additional pre-start cancellation is not yet specified |
| During engine work | Mutator continues handling store changes while worker uses bound C0 | Finish-time live check; changed relevant context: IGNORE, no Seen/proof |
| After engine returns, before Finish is handled | Finish message carries token, not write authority | Mutator rechecks; pruned/replaced token or changed binding: IGNORE |
| Only unrelated snapshot update | Binding and token remain valid | Do **not** discard solely because a new snapshot was published |
| Different validator's valid proof stored while B's earlier claim runs | B's token/context remain valid | B gets its own engine verdict; store first valid proof only (agreed Q14) |

**Open implementation details:** exact context identity and what counts as a material change; how stopped workers are detected without timer-only release; handling enqueue failure/panic/shutdown; bounded pending and verifier work; PR2/PR3 division. See [decisions.md](decisions.md) for agreed status and [report.md](report.md) for the in-depth walkthrough.
