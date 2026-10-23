# Coupled vs. Uncoupled Proof-Context Capture

**Iteration status:** The coupled approach is the previous iteration. Following re-review, the uncoupled approach is the current direction and is being implemented. Implementation is in progress, not claimed complete or verified.

## Why move from coupled to uncoupled?

The coupled implementation addressed a legitimate lifecycle problem:

> A payload's ordinary acceptance marker could survive while its selected payload envelope changed, leaving a prepared proof without a reliable way to establish that its original execution statement was still current.

Its solution was to capture **Q—the complete proof public input—during payload validation**, carry that evidence into Store application, and compare prepared proofs against the currently installed Q.

That strengthened proof freshness, but coupled **ordinary payload processing** to **proof-context acquisition**.

The previous approach is **coupled** because ordinary payload acceptance depended on acquiring proof context. The current approach is **uncoupled** because capture failure cannot change the ordinary payload outcome. This means outcome/failure isolation—not asynchronous execution or zero latency coupling.

The uncoupled approach retains freshness protection while making capture **optional and best-effort**.

## At a glance

| Concern                                           | Coupled approach                                                  | Uncoupled approach                                             |
| ------------------------------------------------- | ----------------------------------------------------------------- | -------------------------------------------------------------- |
| Ordinary state selection                          | Strengthened to obtain trusted exact proof context                | Preserves ordinary payload-validation state selection          |
| Proof-context construction                        | Required before EL notification                                   | Attempted after successful EL notification                     |
| Additional proof-only state lookup/reconstruction | Could be needed                                                   | Prohibited during capture                                      |
| Capture failure                                   | Could produce Ignore, Delay, or a local error before notification | Log locally, carry no evidence, continue ordinary acceptance   |
| Accepted carrier                                  | Required proof-context identity                                   | Carries the envelope and optional Q                            |
| Store application                                 | Applies envelope paired with mandatory context                    | Applies envelope regardless of evidence; installs or removes Q |
| Proof freshness                                   | Current B + full Q comparison                                     | Preserved                                                      |
| Acceptance latency                                | Included proof-context acquisition before notification            | Still includes synchronous capture, but after notification     |

**B** is the beacon block root. **Q** includes the request root, successful-validation assertion, chain ID, and schema ID.

## 1. Mandatory context becomes optional evidence

### Coupled approach

```text
Ordinary validation gates
→ acquire strict exact state
→ construct required Q
→ notify_new_payload
→ return protected accepted result
→ apply envelope and Q
```

If strict state acquisition or Q construction failed, ordinary processing could stop **before the execution engine was notified**:

- Missing state could produce `DelayUntilState`.
- Local lookup or binding failures could produce `Ignore(false)`.
- An Ignore did not itself guarantee a scheduled retry.

These were not demonstrated consensus-safety failures, but introduced proof-related dependencies into ordinary processing.

### Uncoupled approach

```text
Existing ordinary validation
→ notify_new_payload
→ attempt capture
    ├─ success: Some(Q)
    └─ failure: log + None
→ ordinary Accept
→ apply envelope and optional evidence together
```

A returned capture error cannot introduce a payload Reject, Ignore, Delay, or failed application. Existing ordinary validation and engine-submission errors remain unchanged.

## 2. Capture uses already available inputs

Ordinary validation already holds the target block's bid and payload envelope.

The uncoupled approach constructs Q from those inputs **after notification**, without additional state loading, disk access, replay, or proof authentication.

There is an ownership nuance: `notify_new_payload` consumes its payload and parameters. Capture therefore reconstructs equivalent inputs from the retained envelope and bid; it does not borrow the already-consumed arguments.

The implementation must verify that reconstructed inputs match the actual submitted request **field by field**.

## 3. Strict proof preparation is not weakened

Optional capture does **not** mean arbitrary local inputs become trusted proof evidence.

PR17's proof-preparation path still:

- Obtains trusted target-state context.
- Authenticates the signed proof envelope.
- Reconstructs the proof statement using the state-based binder.
- Produces the protected prepared-proof carrier.

The uncoupled approach proposes sharing the **pure request/PublicInput construction helper**, while keeping source selection separate:

- Capture derives commitments from the already held block bid.
- Proof preparation derives them from the trusted target post-state.

Healthy contexts must produce equal Q. A mismatch means the proof is locally inapplicable—not that ordinary acceptance should be reversed. PR2 may refactor PR17-origin code while preserving these guarantees.

## 4. Missing evidence invalidates old authority

Consider replacing a selected envelope for the same B:

```text
Previously selected: envelope A, evidence QA
Newly selected:      envelope B, capture failed
```

The uncoupled approach must apply envelope B **and remove QA**. Keeping QA could preserve authority associated with the previous selection.

Even an equivalent replacement loses proof availability if capture fails: without new evidence, application cannot cheaply prove equivalence. This deliberately favors freshness over availability.

Because the envelope and optional Q travel in one result and apply together, no separate context-publication task or selection-generation protocol is needed.

## 5. What remains unchanged?

The uncoupled approach retains:

- Full-Q content identity, rather than pointer or execution-hash shortcuts.
- Current eligibility and stale-result checks.
- Paired Seen admission and applicable-result completion rules.
- No hashing, replay, authentication, or proof-engine verification on the mutator.
- No PR2 proof-engine scheduling.
- Ordinary cache/storage cleanup.

Physical proof-bookkeeping pruning is deferred to a separate follow-up; correctness invalidation remains required. Historical proof maps may therefore grow temporarily.

## The important limitation

**The uncoupled approach is failure-isolated, not latency-neutral.**

Capture remains synchronous after notification, so construction, copying, and hashing can delay Accept and ordinary application. Notification means **submission**, not an EL `VALID` verdict. Normal-error isolation is not protection against process OOM or unexpected panics.

The improvement is precise:

> Proof-context construction becomes optional bookkeeping that cannot return an ordinary payload failure, while current proof authority remains tied to the selected payload's complete statement.

**Status:** The re-reviewed uncoupled approach is currently being implemented; completion and verification remain pending. See [the detailed design](design-a.md) for interfaces, ownership, replacement cases, and verification requirements. The coupled implementation describes the previous iteration, not the current working tree.
