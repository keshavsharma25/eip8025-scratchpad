# PR2 — Store lifecycle

**Logical stack position, not repository PR #18.** Latest recorded progress: CP1 accepted; CP2 live-applicability proof unresolved. Lifecycle implementation is not claimed.

## Responsibility

Consume PR1’s protected prepared result; define live admission, protected admitted/completion provenance, first-valid storage and consistent finalization pruning. No engine call/dependency or task/message wiring in Store.

## Transition contract

### Admission — one serialized live operation

Check retained target, sufficient current context, vacant verified `(B,T)`, unseen `(B,R)` and unseen `(B,T,V)`. Then insert both keys atomically and return protected admitted input. Any decline inserts neither key. Snapshot prechecks are only an optimization.

This is the accepted semantic direction; CP2 must settle the sufficient context predicate, and CP3 must settle the exact carrier API.

### Completion — applicability before verdict

| Result | Gossip meaning for later integration | Storage / Seen |
| --- | --- | --- |
| Retired or inapplicable context | Local IGNORE | No resurrection; no Seen rollback |
| Separately reported local failure | IGNORE | No verified storage; retain Seen while target retained |
| Applicable engine false | REJECT | No storage; retain Seen |
| Applicable engine true | ACCEPT | Insert only if vacant; preserve first valid bytes; retain Seen |

An occupied storage slot does not suppress an already-admitted applicable attempt’s verdict. It suppresses new admissions. Do not silently replace the admitted statement with a new one.

### Pruning

After finalization/orphan updates, apply the same root predicate to verified proofs and both Seen collections:

```text
keep(B) = B is the latest finalized anchor OR B is retained unfinalized
```

Remove older finalized ancestors and discarded branches. Historical links cannot authorize readmission. A late completion cannot recreate pruned records. Full-payload cache pruning is separate; its residency alone does not define this domain.

## Remaining engineering gates

- Prove cheap current-context evidence, its trust and synchronization across writers/pruners. Prepared B + `PublicInput` alone does not finish this proof.
- Account for equivalent replacement, changed content, cache loss, retirement and unrelated publications without mutator hashing/replay.
- Preserve early empty/type REJECT precedence when adding retirement checks.
- Design narrow admitted/completion construction and consuming engine-input transfer without hidden large proof clones/hashes on the mutator. Protected provenance is not a retry-ownership token.
- Follow local module/visibility conventions; discuss material architecture changes before implementation.

## Required tests

Paired exclusion for same-R/different-V and different-R/same-prover; no partial insertion by losers; invalid/local-failure replay suppression; distinct true/true, true/false and false/true completions; first-valid preservation; stale-context decline; anchor/skipped-slot/orphan retirement; historical-link non-readmission; late-result non-resurrection.

Store tests prove legal transitions, not actual running-service engine-call counts. Admission, completion and pruning stay together in PR2; do not wire a partially enforced lifecycle.
