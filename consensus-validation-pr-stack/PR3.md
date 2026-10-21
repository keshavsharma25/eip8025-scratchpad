# PR3 — task and engine integration

**Logical stack position; repository PR number not assigned here.** Existing plumbing is a skeleton: the enabled task still returns stub IGNORE. The workflow below is the intended design, not running behavior.

## Responsibility

Connect PR1 preparation and PR2 Store transitions through workers/mutator messages. Own engine scheduling, snapshot publication and terminal origin signalling—not Store lifecycle policy.

## Intended flow

1. **Opt-out:** null engine returns IGNORE before preparation, admission or verification.
2. **Prepare worker:** use a coherent snapshot to authenticate and bind; pass the protected result to the mutator.
3. **Live admission:** mutator invokes PR2. A loser terminates without an engine call; a winner schedules engine work without keeping the first worker blocked for a reply.
4. **Engine worker:** consume the exact admitted `ExecutionProof` and transfer its verdict with protected association.
5. **Finish mutator:** invoke PR2 completion; publish snapshots after relevant mutations and deliver one normal-path terminal result to the original gossip/API origin.

## Design rationale and alternatives

- **Non-blocking handoff instead of worker waits:** shared pool capacity should not be occupied while the mutator grants admission.
- **One engine call per admitted attempt instead of mutator re-verification:** keep expensive synchronous work off serialized processing. The trusted orchestration layer must attach the result to the actual admitted input; a boolean by itself is not provenance.
- **Explicit local-failure handling instead of false verdicts:** context/scheduling failures do not justify peer penalties. After admission, failures retain Seen under PR2’s policy, even if engine work never starts.
- **Preserve origin signalling instead of incidental removal:** moving work across stages must preserve the reply destination and avoid duplicate terminal results. Removing `ExecutionProofOrigin` is not an agreed part of this scope.

## Required integration evidence

Use deterministic mock/barrier interleavings to test same-key engine exclusion, independent distinct attempts, stale snapshots, completion after another valid proof, context retirement, scheduling failure, null bypass, snapshot visibility and one normal-path terminal result.

PR2 transition tests cannot establish these runtime call counts or handoffs. A low-priority shared queue is neither verifier isolation nor a concurrency/memory limit. Real-backend interoperability, operational recovery, durable deduplication and new live ingress remain separate workstreams.
