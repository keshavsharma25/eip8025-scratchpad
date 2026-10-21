# EIP-8025 Execution-Proof Networking in Grandine

## Implementation evaluation, consensus-validation prerequisites and expected change footprint

**Purpose:** a standalone technical evaluation of execution-proof gossip support in Grandine's `eth2_libp2p` library, with the prerequisites for a controlled-peer, mock-backed end-to-end demonstration.

**Status:** evaluation and proposed delivery guidance; no networking implementation or executable acceptance result is claimed. Protocol requirements, confirmed service contracts, observed source behavior and recommendations are distinguished below. All source references are public URLs.

## 1. Executive summary

Execution-proof networking should be a compact extension of the existing networking library, not a second proof-validation service. The target is one global `execution_proof` gossip topic carrying `SignedExecutionProofEnvelope` messages. The receiver authenticates the envelope, binds it to locally trusted accepted-payload context, admits the attempt once, verifies it with a controllable mock proof engine, stores the first valid bare envelope and reports the correct gossip outcome.

The library already provides generic publication, decoded-message events, asynchronous validation reporting, fork-topic lifecycle machinery, bounded Snappy transforms and real-node test helpers. The missing work is primarily topic/codec registration, optional enablement, execution-proof awareness in ENRs, subscription/filter integration and tests.

**Expected library-only footprint:** approximately **10–13 files changed, 600–1,100 insertions and 20–60 deletions**, including meaningful tests. A planning midpoint is about **+800 / −40**. This is an estimate, not a measured implementation diff or delivery commitment. It excludes the Grandine P2P bridge, consensus-validation PR1–PR3, proof backends, startup integration outside the library and unrelated dependency updates.

Networking work can proceed alongside the Store-lifecycle implementation. End-to-end enablement must wait for the legal Store transitions and service orchestration to be complete and verified. A working codec or a received-message log is not evidence that proof validation and storage are safe.

A baseline refresh also matters: the latest inspected public PR1 changes binding to a **zkBoost-compatible request root**. Earlier lifecycle work was based on an earlier PR1 revision. Their statement-identity and binding conventions must be aligned before integration; source similarity to another client does not establish interoperability.

## 2. Scope and target outcome

### 2.1 Agreed outcome

The first end-to-end milestone is a **controlled-peer, mock-backed gossip demonstration**:

1. An external controlled peer publishes a signed execution-proof envelope.
2. Grandine receives it through the actual libp2p gossip path.
3. Real preparation checks authenticate the envelope and reconstruct its trusted statement.
4. Live Store admission authorizes the attempt and consumes both duplicate keys atomically.
5. A controllable mock proof engine returns a definitive valid or invalid verdict.
6. Applicable completion stores the first valid bare envelope and preserves the selected Seen lifecycle.
7. Gossip receives the appropriate ACCEPT, REJECT or IGNORE result.
8. An observer can demonstrate forwarding after acceptance, rather than only initial receipt.

The mock replaces cryptographic proof verification, not envelope authentication, trusted-context loading, binding, admission or completion. The result is a network/service-mechanics prototype, **not real-proof interoperability**.

### 2.2 Library delivery versus complete demonstration

| Boundary | Responsibility | Included in library diff estimate? |
| --- | --- | --- |
| Networking library | Topic, codec, transport bounds, subscription lifecycle, ENR awareness, generic delivery/reporting and library tests | Yes |
| Grandine P2P bridge | Submit decoded signed envelope plus origin identity to the controller; carry results back to gossip | No |
| Consensus-validation service | Preparation, live admission, engine scheduling, exact completion, storage, snapshots and pruning | No |
| Node startup | Select/inject the mock or null engine and align enabled configuration across layers | Only the library-facing configuration portion |
| Full demonstration | Coherent block/payload/state fixtures, eligible signer, connected peers and observable outcomes | Library transport tests only; complete demonstration is separate integration work |

### 2.3 Explicit boundaries

The target uses a single global topic and accepted-payload-dependent preparation. It does not introduce proof generation, proof-specific Req/Resp, historical proof serving, additional HTTP ingress, proof-count thresholds, proof-driven consensus effects, durable restart deduplication or generalized recovery/token infrastructure. Real-backend integration and independent statement vectors remain separate workstreams; they do not block the selected mock-backed milestone or become extra PR2/PR3 scope.

## 3. Reference baselines and evidence

| Reference | Inspected revision | Meaning and limit |
| --- | --- | --- |
| [Networking library][net-repo] | [`55cdfba`][net-pin] | Local `main` snapshot used for the library architecture and diff estimate; not a claim that remote `main` cannot move |
| [Grandine integration source][grandine-pin] | `80ed3be0` | Refreshed local HEAD and public PR1 head; earlier observations at `a5f27d52` are not the current PR1 source baseline |
| [Grandine embedded networking revision][embedded-net] | `d03b1685` | Older library revision recorded by the inspected Grandine commit |
| [Consensus-specs local comparison revision][spec-local] | `460fd37c3` | Feature networking, beacon-chain and fork-choice files have no committed difference from the selected protocol pin |
| [Selected protocol reference][spec-p2p] | `12c8b24b` | Authority for gossip order, envelope rules and the global topic |
| [Prysm reference implementation][prysm-pin] | `e99a87cf` | Optional-proofs implementation inspected as a comparison, not a conformance oracle |

The library inventory covers **48 Rust source files, approximately 27,700 source lines and four integration-test Rust files**. Topic/message references and relevant control paths were traced across modules. This is not a line-by-line correctness audit of every RPC and peer-management implementation.

Public metadata confirms [PR1 / repository PR #17][pr1] is open and unmerged at the inspected head. PR2 and PR3 below are logical positions in the consensus-validation stack, not repository PR numbers. Their remaining contracts are prerequisites, not claims that those implementations are published or complete.

## 4. Protocol requirements

### 4.1 Topic and wire object

The target topic is:

```text
/eth2/{fork_digest}/execution_proof/ssz_snappy
```

The object is `SignedExecutionProofEnvelope`, containing a bare `ExecutionProofEnvelope`, validator index and BLS signature. The bare envelope carries opaque bounded proof bytes, a proof type and the target beacon block root. The execution-proof message does not supply authoritative public input: the receiver reconstructs that input from trusted local context.

The [EIP-8025 networking specification][spec-p2p] adds the global topic and awareness field. The [beacon-chain specification][spec-beacon] defines the containers, supported-type checks, signature validation and binding. The inherited [gossip encoding rules][spec-encoding] use **SSZ followed by Snappy block compression**, not the streaming Snappy format used by Req/Resp.

### 4.2 Size limits

| Layer | Required bound or behavior |
| --- | --- |
| Outer gossipsub RPC message | Existing inherited maximum including framing, control messages and topics |
| Compressed message payload | Existing global compressed-payload limit |
| Uncompressed message payload | Lower of the global maximum and the message's type-specific SSZ maximum |
| Signed execution-proof envelope | **4,194,449 bytes** maximum |
| Opaque proof data | **4,194,304 bytes / 4 MiB** maximum |

The extra 145 bytes in the envelope maximum account for the signed wrapper and bare-envelope fixed fields/SSZ offsets. These are encoded-size bounds, not estimates of total in-memory allocation. See the [inherited size rules][spec-size] and [Grandine proof constants][grandine-consts].

Bounds must cover inbound messages and invalid outbound publication. A bounded `ProofData` decoder does not replace compressed/decompressed transport checks, and a global transport check does not replace the tighter type-specific maximum.

### 4.3 Ordered gossip validation

| Order | Check | Failure classification | Primary service owner |
| --- | --- | --- | --- |
| 1 | Proof data is nonempty | REJECT | PR1 |
| 2 | Proof type is supported | REJECT | PR1 |
| 3 | Target beacon block is known | IGNORE | PR1 |
| 4 | No valid proof is already stored for target block and type | IGNORE | PR1 snapshot check; PR2 live recheck |
| 5 | Bare-envelope root has not been processed | IGNORE | PR1 snapshot check; PR2 live admission |
| 6 | Prover has not attempted this block/type key | IGNORE | PR1 snapshot check; PR2 live admission |
| 7 | Accepted payload and trusted preparation context are available | IGNORE | PR1 |
| 8 | Envelope authentication succeeds | REJECT | PR1 |
| 9 | Reconstruct and bind the exact expected engine input | Protected preparation; local failures are not automatic REJECT | PR1 |
| 10 | Mark authenticated proof and prover attempt Seen | Atomic live transition in the asynchronous implementation | PR2 |
| 11 | Proof engine verifies the exact admitted input | REJECT on definitive false; local inability is not a false verdict | PR3 execution; PR2 completion |

The ordered rules come from [gossip validation][spec-gossip]. The live admission/completion checks are the selected implementation's factoring of those sequential rules across asynchronous workers and the serialized mutator. Snapshot prechecks alone never authorize engine entry.

### 4.4 Discovery awareness

The ENR field is **`eproof`, typed as `Uint8`**. A present, nonzero value means execution-proof awareness. An aware node subscribes to the topic and implements its validation rules. Missing or zero does not indicate awareness. Clients **MAY** prefer proof-aware peers; a dedicated proof-peer maintenance algorithm is not mandatory for the controlled-peer milestone.

Awareness is not an authorization token, validator identity, supported-type bitmap or promise of real-proof interoperability. The absence of awareness is not itself a protocol-invalidity reason for an otherwise valid message. See [execution-proof awareness][spec-awareness].

### 4.5 Downstream processing

The [fork-choice handler][spec-storage] verifies and stores a bare envelope per block/type without changing head selection, fork-choice weight, beacon state or Gloas payload status. Grandine's selected implementation factors verification into protected worker execution and Store completion rather than rerunning expensive work on the mutator.

No EIP-8025-specific Req/Resp protocol is defined. An unknown-block IGNORE does not itself create a requirement for proof fetching, buffering or automatic retry.

## 5. Current networking architecture

The relevant reusable pieces are already present:

| Existing mechanism | Source | Consequence |
| --- | --- | --- |
| Typed gossip topic names, parsing/display and fork-topic selection | [Topics][net-topics] | Add one global kind rather than a new topic framework |
| Typed pubsub messages and SSZ encode/decode | [Pubsub codec][net-pubsub] | Add a signed-envelope variant and follow nearby Gloas decode conventions |
| Compressed and advertised decompressed-length checks | [Snappy transform][net-pubsub] | Reuse bounded transport; add the proof-specific typed maximum |
| Generic publication and `NetworkEvent::PubsubMessage` delivery | [Network service][net-service] | Existing event includes message ID, propagation source, topic and decoded message |
| Deferred ACCEPT/IGNORE/REJECT reporting | [Network service][net-service] | Existing reporting API controls propagation after application validation |
| Fork subscriptions and old-digest unsubscription | [Network service][net-service] | Extend existing lifecycle; do not add a parallel timer system |
| Topic whitelist and subscription count limits | [Service utilities][net-utils], [network service][net-service] | A topic must be recognized by filters and accounting, not just the parser |
| ENR construction and persisted-record comparison | [ENR support][net-enr] | Fresh creation and restart behavior both need awareness handling |
| Scoring and generic topic metrics | [Scoring][net-scoring], [metrics][net-metrics] | Explicitly assess proof-topic treatment; many counters already generalize |
| Real-node pairs and linear topology helpers | [Integration-test helpers][net-test-common] | Network tests can extend the current harness |

The generic event and validation-result interfaces mean a new proof-specific libp2p behaviour, transport or event channel is not expected. Existing [network configuration][net-config] already enables application validation before propagation and uses anonymous gossipsub validation mode. The validator's BLS envelope signature is a separate application-level identity check; it is not supplied by libp2p peer authentication.

A large fraction of the repository is RPC or peer management. Those areas are not new implementation targets merely because this feature adds a large message type.

## 6. Current gaps

At the inspected networking baseline, there is no execution-proof topic kind or signed-proof pubsub variant. There is also no proof-aware enablement path in topic configuration, no `eproof` ENR support and no proof-specific network test suite.

The expected gaps are:

- Topic identity, parsing, display and classification as a non-indexed global topic.
- Signed-envelope pubsub kind, encode/decode and concise display output.
- Proof-specific size checking and tests for malformed/oversized input.
- Feature-controlled subscription selection and consistent fork lifecycle.
- Topic whitelist/count/metrics enumeration coverage.
- Fresh and persisted ENR awareness handling.
- Deliberate topic-scoring treatment and honest failure reporting.
- Real transport tests for receipt and application-controlled forwarding.

Separate gaps remain outside the library: the controller ingress bridge, legal live Store transitions, actual proof task integration, snapshot publication and complete demonstration fixtures. The inspected [proof task][grandine-tasks] still returns an enabled-path stub IGNORE. Existing [mutator signalling][grandine-mutator] is scaffolding, not completed verified-proof storage.

## 7. Consensus-validation prerequisites: PR1, PR2 and PR3

### 7.1 Stack summary and readiness

| Stage | Required contract | Current status and networking consequence |
| --- | --- | --- |
| **PR1 — preparation** | Ordered prechecks, exact trusted target state and accepted payload, envelope authentication, statement binding and protected prepared input with cached bare-envelope root | [Published as repository PR #17][pr1], open/unmerged at `80ed3be0`. Preparation is implemented; it does not consume Seen, execute the engine, store proofs or return final gossip ACCEPT |
| **PR2 — Store lifecycle** | Cheap live context/retention checks; protected admitted/completion provenance; atomic paired permanent-Seen admission; applicable first-valid completion; shared pruning | In progress. Initial contract accepted; live-context work is implemented/in review on an earlier base. Final acceptance, protected lifecycle APIs, admission/completion/pruning and accumulated lifecycle verification remain pending |
| **PR3 — orchestration** | Nonblocking worker/mutator handoffs; engine execution only after live admission; exact result transfer; snapshot publication; exactly one normal-path terminal origin result and disabled bypass | Skeleton exists, but the enabled task remains a stub. Full running-service guarantees remain a prerequisite for end-to-end enablement |

Reported focused Store results exist for the in-progress lifecycle context work, but were not rerun for this evaluation and do not establish complete PR2 acceptance. Neither PR1 publication nor an intermediate Store test suite establishes a running proof-gossip service.

### 7.2 Confirmed contracts networking must preserve

| Contract | Why the networking integration depends on it |
| --- | --- |
| Authenticate and bind before admission | A forged envelope must not consume another validator's attempt key |
| Check all live predicates before inserting either Seen key | Concurrent stale snapshots must not cause duplicate authorization or partial bookkeeping |
| Permanent Seen until target retirement | Definitively invalid attempts and local/non-start failures do not release their admitted keys |
| Exact input/result provenance | A caller cannot manufacture verified storage with arbitrary bytes and a supplied boolean |
| Distinct attempts may overlap | No new exclusive block/type in-flight gate; both duplicate keys must differ |
| First-valid storage never overwrites | Later admitted valid attempts can ACCEPT without replacing the stored bare envelope |
| Existing storage does not erase an admitted applicable verdict | A later applicable false verdict remains REJECT even when another proof won storage |
| Shared retained-root lifetime | Storage and both Seen collections retire together; late results cannot resurrect a retired target |
| Expensive work remains off-mutator | No proof hashing, BLS, replay/materialization, payload rehash or engine call on the serialized path |
| Proofs have no consensus consequences | Gossip results must not alter payload validity, head/weight or beacon-state transitions |
| Typed expected races/local failures | Duplicates, context loss and retirement are ordinary declines/discards, not panics or generic peer rejection |

Here `B` is the beacon block root, `T` the proof type, `V` the validator index and `R` the bare-envelope root. Retention is root-based: keep the latest finalized anchor and unfinalized retained roots, not an independent time-based proof-cache window.

### 7.3 Fresh binding/base-alignment gate

The refreshed [PR1 binding change][grandine-binding-change] uses `zkboost_compatible_root()` for `PublicInput.new_payload_request_root`, rather than the canonical progressive-container root. The public [container definition][grandine-containers] also documents progressive versioned hashes and the selected construction bound. This is a deliberate compatibility target, not pure protocol-pin SSZ equivalence.

Before PR2/PR3 integration, establish that preparation, accepted-payload identity publication and completion compare the same **B + full PublicInput** under this binding convention. Reconcile the earlier lifecycle base with the refreshed PR1; this report does not perform a rebase or assume equivalence. Do not compare a prepared identity to itself and call that a live-context check.

This update removes the basis for an unconditional claim that Grandine and Prysm use different *outer request-root* conventions. It does not prove identical nested encodings, guest versions, keys, public-input conventions or external fixture compatibility. Those questions remain outside the mock-backed networking milestone.

## 8. End-to-end message and ownership flow

```text
Controlled publisher
    |
    | signed envelope, SSZ + Snappy, correct fork topic
    v
eth2_libp2p
    transport bounds -> topic/SSZ decode -> generic message event
    |
    | signed envelope + propagation source + message ID
    v
Grandine P2P bridge / controller
    |
    v
Worker: PR1 prepare/authenticate/bind
    |
    | protected prepared input; no Seen writes or engine call
    v
Mutator: PR2 live applicability + paired admission
    |
    | publish bookkeeping snapshot; transfer exact admitted input
    v
Worker: PR3 run mock ProofEngine once for this admitted attempt
    |
    | exact corresponding result, retained origin ownership
    v
Mutator: PR2 completion / first-valid storage
         PR3 snapshot publication + terminal signalling
    |
    v
P2P bridge -> eth2_libp2p validation report
    ACCEPT: eligible for gossip forwarding
    IGNORE: withhold forwarding without declaring peer invalidity
    REJECT: protocol-invalid message / applicable definitive invalid proof
```

The admission handoff must not leave a worker blocked waiting for a mutator response. Engine scheduling occurs only after live admission and the required bookkeeping publication. Scheduling failure is a local outcome; permanent Seen remains consumed after admission even if verification never starts.

### Four identities that must not be confused

| Identity | Purpose |
| --- | --- |
| Gossip origin (`source`, `message_id`) | Report the result to the correct pending gossip message; source is the forwarding peer |
| Root key `(B,R)` | Suppress the same bare envelope even if signed by different validators |
| Prover key `(B,T,V)` | Suppress changed proof bytes from the same validator for the same block/type |
| Storage key `(B,T)` | Retain the first valid bare envelope, independently of Seen attempt memory |

The public-input identity is a further applicability/provenance fact, not a replacement for any of these keys. Gossipsub's duplicate cache does not enforce either semantic admission key. A forwarding peer and the signed-envelope validator are not necessarily the same participant.

## 9. Required networking changes

### 9.1 Topic and codec

Extend [topic support][net-topics] with the constant/kind, parser/display handling, non-indexed classification, subscription selection and all-topics enumeration. The optional topic must be included in worst-case subscription/metrics accounting even when normal feature enablement defaults off.

Extend [pubsub support][net-pubsub] with an `Arc`-backed signed-envelope variant following nearby conventions, kind selection, SSZ encode/decode and short metadata-only display. Check the type-specific maximum before typed decoding and apply the selected Gloas/fork-digest restrictions. Keep support checks in the intended validation layer: bounded SSZ decoding and runtime supported-type validation are different concerns.

### 9.2 Configuration and lifecycle

Propagate a single truthful library-facing enablement decision through [network configuration][net-config], [topic configuration/globals][net-globals] and ENR construction. Disabled-by-default is recommended; the exact public option name is an implementation choice. Outer node startup must supply that decision only when the selected service configuration is usable.

Use existing subscribe-new-fork and unsubscribe-old-fork paths. Test enablement across fork transitions, not only initial subscription. Manual extra-topic configuration must not silently make an otherwise disabled node claim full proof-service readiness. Dynamic runtime toggling is not required for the first milestone; restart consistency is required.

### 9.3 Filters, enumeration and publication cache

Update the [whitelist][net-utils] and any exhaustive topic matches. Reuse generic metrics where sufficient. The [outbound gossip cache][net-gossip-cache] is a retry cache for failed publication, not semantic Seen and not a proof-validation retry mechanism. Handling the new enum variant does not automatically require enabling retries; if enabled, its byte retention needs an explicit bound/policy.

### 9.4 ENR awareness and persistence

Extend [ENR support][net-enr] to construct and read the typed awareness value. Missing/zero must remain distinct from nonzero awareness. Inspect `use_or_load_enr` and `compare_enr`: a persisted record must not retain enabled awareness after disabled startup, or suppress awareness after enabled startup. Test changed awareness causing the normal record replacement/sequence lifecycle rather than trusting fresh-record tests alone.

Prefer local `Eth2Enr`/ENR-accessor conventions. Do not introduce a parallel capability registry or new peer database solely for the controlled-peer demo.

### 9.5 Integration APIs

Reuse generic publication, receive events and result reporting in the [network service][net-service]. The outer [Grandine network bridge][grandine-network] is expected to dispatch proof messages with a `GossipId` and reuse the existing return channel. It is a separate change outside the library-only estimate. Preserve the existing origin contract unless its simplification is separately agreed.

No new dependency is expected: `types`, SSZ, Snappy, ENR, metrics and libp2p are already dependencies. The library [manifest][net-manifest] inherits workspace dependencies, so a compatible Grandine workspace containing the EIP-8025 types is a build prerequisite; this extracted repository is not an independent build environment by itself.

### 9.6 Minimal diagnostics

Distinguish transport receipt, decode failure, preparation decline, authentication rejection, live admission, engine verdict, local scheduling/capacity failure, completion discard, verified storage and terminal gossip reporting. This is a cross-layer diagnostic contract: not every counter belongs in the library. Reuse the existing metrics/logging frameworks and avoid raw proof bytes, signatures or high-cardinality block/validator labels in metrics. A receipt log, engine-success log and stored-envelope assertion establish different facts. Retain enough bounded reason information to distinguish peering/subscription failures from missing context, signing/binding failures and local overload.

## 10. Failure classification and peer scoring

| Situation | Intended semantic outcome | Important qualification |
| --- | --- | --- |
| Malformed topic/SSZ or exceeded protocol size bound | Transport/decode rejection | Demonstrate bounded handling and no service/engine entry |
| Empty proof or unsupported type | REJECT | Preserve early precedence; unknown target does not override these checks |
| Unknown block, unavailable trusted context or payload | IGNORE | No accusation of invalid proof and no fabricated context |
| Duplicate root/prover key or already stored proof at new admission | IGNORE | Decline without partial Seen insertion |
| Invalid envelope signature/authentication | REJECT | No admission and no Seen consumption |
| Admitted, applicable engine false | REJECT | Keep Seen; a concurrent stored winner does not erase this verdict |
| Admitted, applicable engine true | ACCEPT | Store only if vacant; preserve a prior valid winner |
| Context mismatch or retirement before completion | IGNORE | Check applicability before using the engine verdict |
| Local overload, disabled service, scheduling failure or unavailable verifier configuration | Local IGNORE/decline as appropriate to the layer | Not evidence that the sender supplied an invalid proof |

The existing [mutator skeleton][grandine-mutator] has a generic error-to-rejection branch. Actual integration must not let local errors fall into that branch indiscriminately. The boolean `ProofEngine` boundary remains unchanged: local scheduling/configuration failures must not be encoded as an engine `false` verdict.

[Library scoring][net-scoring] currently has explicit parameters for only a subset of gossip kinds. Decide proof-topic treatment deliberately. Do not blindly copy block delivery-rate expectations or claim measured proof rates. An explicitly limited/unscored initial policy versus conservative proof-specific parameters remains an implementation choice; peer-outcome reporting must still remain honest. Default-disabled scoring or trusted peers in a test cannot demonstrate scoring correctness.

A validation timeout also does not prove that already-running verification stopped. Transport message expiry, service completion and permanent Seen are separate lifecycles. The test suite should cover delayed results and record whether the pending gossip message still exists when reporting occurs.

## 11. Resource bounds and operational safety

### 11.1 Bound each resource at its owner

| Resource | Expected owner | Requirement before enabled exposure |
| --- | --- | --- |
| Encoded frame and compressed/uncompressed bytes | Library/transport | Enforce existing transport maxima and proof-specific SSZ maximum |
| Decoded envelopes awaiting service work | P2P/service ingress | Finite count/byte/work admission; avoid an unbounded large-message backlog |
| Preparation work, including hashing/BLS/replay | Service scheduling | Finite work budget; pre-admission duplicate work can still occur |
| Admitted engine calls | Service/engine scheduling | Finite concurrency; distinct semantic attempts are permitted but not unlimited work |
| Outbound publication retries | Library cache if enabled | Deliberate lifetime/capacity; not an accidental large-proof memory reservoir |
| Seen and verified proof records | Store lifecycle | Shared retained-root pruning; this is not a hard memory cap during non-finality |
| Pending validation/forwarding state | Gossipsub plus service handoff | Align queueing/verification latency with actual pending-message retention |

No numeric capacity is asserted as safe by this evaluation. Choose and document small finite prototype values with saturation/progress tests. Their implementation outside the library is not included in its diff estimate. If the existing ingress path cannot provide bounded handoff, that is an enablement blocker rather than a reason to widen the estimate silently.

### 11.2 Important non-guarantees

- Low-priority scheduling is neither worker isolation nor a concurrency cap.
- Semantic Seen suppresses repeated keys, not arbitrary distinct proofs or pre-admission BLS work.
- Finalization-driven retention is not bounded-memory protection during stalled finality.
- A timeout can stop waiting without interrupting a native call.
- An `Arc` avoids some copies but does not prove zero-copy transport or bounded retained bytes. The current publication path and optional retry cache already clone buffers; account for those copies before making performance claims.
- Basic finite admission is required for the prototype; production-scale tuning, crash recovery and durable attempt storage are not implied.

### 11.3 Decompression strategy

The existing [Snappy transform][net-pubsub] checks compressed length and advertised decoded length before allocating decompressed output. Reuse that global bound and the existing typed size helper first. A proof-specific preallocation check could tighten memory further, but a generalized per-topic compression framework is not presumed necessary. The acceptance argument must name the maximum allocation allowed before typed rejection and test compression-bomb/malformed-length cases.

### 11.4 Cost model and retained memory

For an envelope of S bytes, decoding, decompression, hashing and full-buffer copies are at least linear in the bytes processed; a small enum diff does not make large-message handling cheap. Live admission should use cached identity/root data and expected-average keyed lookups, while accounting separately for persistent-collection copy-on-write costs. Shared retirement pruning scans the relevant collections. Seen memory grows with retained distinct admitted attempts, and verified storage additionally retains proof bytes; stalled finality can extend those lifetimes. Engine latency is backend-dependent and unmeasured here. These are qualitative implementation obligations, not measured constant-time latency or DDoS-resistance claims.

## 12. Prysm comparison and lessons

### 12.1 What its networking path demonstrates

| Prysm mechanism | Public source | Useful lesson |
| --- | --- | --- |
| Global topic name and signed-envelope mapping | [Topics][prysm-topics], [type mapping][prysm-mappings] | Reuse existing gossip registration/encoding infrastructure |
| Feature-gated validator/subscriber registration | [Subscription setup][prysm-subscription] | Optional participation must connect validation and downstream handling |
| Typed `eproof` and remote awareness reader | [Awareness helper][prysm-awareness] | Present/nonzero awareness, not a subnet bitmap or authentication mechanism |
| Enabled ENR advertisement | [Discovery construction][prysm-discovery] | Advertisement and configured participation must agree |
| Generic compressed/decompressed bounds | [Encoder][prysm-encoder] | Size safety is layered rather than delegated only to proof parsing |
| Ordered validation and Seen before verification | [Proof validator][prysm-validator] | Authenticate before consuming an attempt; distinguish invalidity from inability |
| Accepted-object transfer through `ValidatorData` | [Proof validator][prysm-validator], [subscriber][prysm-subscriber] | Retain a clear handoff between validation and recording |
| Topic scoring registration | [Scoring][prysm-scoring] | Include the topic, but its block-parameter reuse is marked TODO rather than measured guidance |
| Semaphore-limited verifier entry | [Proof engine][prysm-engine] | Active verification needs a finite budget independent of duplicate keys |

Prysm reuses its internal subnet discovery machinery with a synthetic index `0` to find participants in this single global topic. Its source explicitly calls that a workaround. The wire topic remains `execution_proof`, with no index suffix. Grandine does not need to import this workaround or introduce a new `Subnet` variant for the global topic.

### 12.2 Differences that must not be copied blindly

- **Atomicity:** Prysm separately looks up two caches, authenticates and adds the keys. This does not establish compound atomic admission if callbacks overlap. A source-level interleaving is possible; no runtime race reproduction is claimed. Grandine's live serialized paired admission remains necessary.
- **Retention:** Prysm uses two independently capacity-evicted Seen LRUs, each sized at 1,000 in [service initialization][prysm-sync-service]. That is not Seen-until-retirement and is not a measured Grandine capacity recommendation.
- **Authentication context:** [Prysm envelope preparation][prysm-chain] uses head state with the target block's epoch. Grandine preserves exact target-state preparation and live statement applicability.
- **Storage:** Prysm records verified-type flags. Grandine must retain the first valid bare envelope, not just a flag.
- **Consensus policy:** Prysm's receive path can participate in proof-count-dependent payload validity. That behavior is outside this target; proofs here have no consensus consequences.
- **Binding:** [Prysm's request-root implementation][prysm-request-root] deliberately uses a zkBoost-compatible outer root. Refreshed Grandine PR1 now also selects a compatible root, but end-to-end compatibility has not been demonstrated. Do not reuse a foreign fixture as an unquestioned oracle.
- **Engine API/cancellation:** Prysm returns errors distinguishing invalid from operational failure and can cancel semaphore waiting. Grandine currently has a [consuming boolean engine interface][grandine-engine]. The comparison does not authorize a new error/cancellation API or prove interruption of running verification.
- **Completion:** Prysm's accepted-type recording is not evidence for Grandine's full context-sensitive completion and retirement contract.

### 12.3 Test evidence limits

The [Prysm validator tests][prysm-validator-tests] cover sequential outcomes such as valid, syncing, empty/unsupported proof, unknown block, existing proof type, duplicates, unavailable payload, invalid authentication, invalid proof and local verification failure. The invalid-proof test also checks changed bytes under the same prover key are ignored afterward.

These tests use a mocked chain service and placeholder signatures. They are useful routing/outcome tests, not a complete real-signature, trusted-state, concurrent-admission or three-node propagation demonstration. The local-failure test alone does not prove a retry policy; retention follows from inspected source. No Prysm test was executed for this evaluation.

## 13. Test strategy and acceptance criteria

### 13.1 Library unit and bounded-codec tests

| Test group | Required evidence |
| --- | --- |
| Topic identity | Exact global name, parser/display round-trip, no subnet identity, valid encoding and digest shape |
| Fork/enablement | Disabled absence; enabled selected-fork presence; wrong/unknown digest behavior; new-fork subscription and old-topic removal |
| Typed codec | Signed-envelope SSZ round-trip with representative and maximum-size proof data; exact fields preserved |
| Size boundaries | Maximum signed size accepted as a size boundary; maximum + 1 rejected; malformed offsets/oversized proof list rejected |
| Compression | Malformed Snappy, advertised decompressed oversize, compressed oversize and normal round-trip; bounded allocation argument |
| Whitelist/accounting | Recognized proof topic, sufficient worst-case topic count, optional-feature defaults do not omit capacity accounting |
| ENR | Absent, zero and nonzero values; malformed value handling; enabled/disabled fresh construction; persisted-awareness changes and record lifecycle |
| Scoring/reporting | Selected topic policy is installed where intended; ACCEPT/IGNORE/REJECT are not conflated |
| Publication cache | New kind handled without panic; no unintended retry retention; configured retry behavior tested if enabled |

A maximum-sized zero-filled buffer that merely passes the size guard is not a successful typed decode test. Include valid constructed envelopes at the boundary as well as guard-precedence tests. Codec tests need not use cryptographically valid signatures; service integration tests do.

### 13.2 Actual library network tests

Extend the existing [real-node test helpers][net-test-common] and explicitly register a gossip test module through the [integration-test entrypoint][net-test-main]. Prefer bounded waits/event-driven readiness over arbitrary sleeps.

1. Establish compatible fork/configuration and observed subscriptions before publication.
2. Publish a signed-envelope-shaped message and assert the receiving generic event preserves its fields and origin identity.
3. Report ACCEPT and observe forwarding in a publisher → receiver → observer topology with no direct publisher–observer route.
4. Report IGNORE and REJECT in separate cases and assert no forwarding over a bounded observation interval. Use fresh message identities to avoid duplicate-cache false positives.
5. Exercise disabled participation, malformed/oversized delivery where the harness permits, and fork-topic lifecycle.
6. Keep at least the relevant scoring checks free of trusted-peer/scoring-disabled bypasses.

These tests deliberately supply application outcomes; they prove transport/reporting, not that consensus validation actually ran. Anonymous propagation, exact duplicate suppression and test topology must be accounted for when interpreting observations.

### 13.3 Full-service acceptance outside the library diff

| Scenario | End-to-end invariant |
| --- | --- |
| Valid signed envelope + mock true | Real preparation/admission; one call for the admitted same-key attempt; first valid bare bytes stored; ACCEPT |
| Bad signature | REJECT; neither Seen key consumed; zero engine calls |
| Authenticated mock false | REJECT; no verified insertion; both admitted keys remain |
| Unknown block/unavailable context | IGNORE; no fabricated statement or engine entry |
| Same R / different V and different R / same K | Live exclusion against stale snapshots; loser adds neither key |
| Distinct R and K for one block/type | Both may admit subject to global capacity; preserve each applicable verdict |
| Two true completions, either order | Both applicable verdicts ACCEPT; first stored bytes never change |
| Mixed true/false completions | False remains REJECT even after another valid winner |
| Retire between prepare/admit/complete | No readmission, Seen recreation or late storage resurrection |
| Context changes versus equivalent allocations | Changed statement declines/discards; equivalent content is not invalidated solely by allocation identity |
| Local overload/non-start failure | Non-punitive local outcome; post-admission Seen remains consumed |
| Disabled/null configuration | Bypass before preparation/admission/engine work; no misleading awareness advertisement |
| Saturation | Finite queued work/bytes and engine calls; ordinary node work continues |
| Gossip expiry versus delayed result | Honest reporting/diagnostics; no implied cancellation or duplicate terminal result |

### 13.4 Verification procedure

Run library unit/integration tests and affected Grandine checks in a compatible workspace using its pinned toolchain, actual scripts and backend feature selection. Recheck the [workspace manifest][grandine-manifest] and [CI lint script][grandine-clippy] at implementation time rather than treating guessed command lines as a harness.

Record exact revisions, feature flags, commands, exit codes, discovered/executed test counts, warnings and fixture provenance. Distinguish pre-build dependency failure, zero matching tests, failing tests and successful execution. Add focused tests with each implementation change; formatting alone is not acceptance. Review architectural fit, ownership and failure classification in addition to passing tests.

## 14. Expected diffstat and change footprint

### 14.1 Library-only estimate

**Planning range: 10–13 files, +600–1,100 / −20–60. Midpoint: approximately +800 / −40.** Most deletion should be small enum/configuration adjustments, not broad replacement. Approximately 200–350 production additions are expected, with the remainder predominantly tests/fixtures. The ranges are intentionally approximate and are not independent promises that every category reaches its upper bound.

| Area | Likely source touchpoints | Estimated additions, including related tests |
| --- | --- | ---: |
| Topic/subscription identity | [Topics][net-topics] | 60–110 |
| Pubsub codec, bounds and codec tests | [Pubsub][net-pubsub] | 140–230 |
| Library enablement propagation | [Configuration][net-config], [globals][net-globals] | 20–45 |
| Awareness and ENR restart correctness | [ENR support][net-enr] | 120–190 |
| Whitelist/cache exhaustive integration | [Utilities][net-utils], [gossip cache][net-gossip-cache] | 20–50 |
| Scoring/service adjustments | [Scoring][net-scoring], possibly [service][net-service] | 30–70 |
| Real network tests and small harness changes | [Existing helpers][net-test-common], [entrypoint][net-test-main], new test module | 200–350 |

The category sum is roughly 590–1,045 additions; the headline rounds this into a practical range with modest integration allowance. New proposed files have no source link until implemented; existing linked files show the conventions they should follow.

### 14.2 Historical calibration

The public [execution-payload-bid topic addition][net-topic-history] changed **four files, +61 / −1**. A separate [Gloas whitelist follow-up][net-whitelist-history] added four lines. These show that topic/variant plumbing alone is small and that filter integration can be overlooked.

They do not include this feature's optional awareness contract, ENR persistence coverage or complete propagation tests. Therefore a roughly 100-line topic patch is plausible as an intermediate skeleton, but not evidence of the evaluated tested deliverable.

### 14.3 Exclusions and estimate risks

Not counted: Grandine controller/P2P bridge, PR1–PR3, mock-engine startup outside the library, service queue budgets, real verifier integration, independent vectors, unrelated library-main backports, workspace dependency reconciliation or full demonstration tooling.

Preferential automatic discovery/minimum-proof-peer maintenance, generalized topic-aware compression, dynamic runtime toggling or a substantial new test harness can exceed the range. If the diff exceeds approximately 1,500 additions, review the reason for scope growth rather than treating that number as a hard correctness limit. A smaller diff is not better if it omits safety or tests.

## 15. Suggested delivery sequence and dependency gates

This is proposed sequencing, not authorization to implement or publish an incomplete running service.

| Stage | Deliverable | Dependency / gate |
| --- | --- | --- |
| 1. Freeze integration contract | Agree wire/container baseline, binding convention, library enablement, result classifications and compatible workspace | Refresh source/PR state and reconcile earlier lifecycle work with current PR1 |
| 2. Add bounded transport support | Topic/codec/filter changes with unit and bounds tests | Existing EIP-8025 types; can proceed alongside unfinished PR2 |
| 3. Add awareness/lifecycle | Configuration, enabled subscription, ENR persistence and fork tests | Truthful capability contract; no default live exposure |
| 4. Prove library transport/reporting | Real-node receive and application-controlled propagation tests | Controlled peers; application outcomes supplied by the test |
| 5. Complete service prerequisites | PR2 legal transitions and PR3 orchestration, with interleaving/terminal-result tests | Accepted context/provenance/retention contracts and required checks |
| 6. Bridge and bound | Submit messages/origins, report typed results, install finite work limits and aligned startup | Complete service; no generic local-error → REJECT path |
| 7. Demonstrate milestone | Real signing/preparation/lifecycle + mock verdict + stored bytes + observer outcome | Reproducible fixtures, topology, negative cases and verification evidence |

The library's newer `main` includes fork-subscription, processor-limit and dependency backports absent from Grandine's older embedded revision. Keep any submodule/dependency update explicit and independently review its compatibility. Do not include unrelated backport churn in the proof-networking diffstat or assume arbitrary copying between the two revisions is safe.

## 16. Risks, open implementation choices and non-goals

### 16.1 Confirmed versus still to decide

**Confirmed:** global gossip; accepted-payload-dependent preparation; controlled-peer mock milestone; PR1/PR2/PR3 responsibility split; authenticated paired permanent-Seen admission; distinct-attempt overlap; first-valid bare storage; shared retirement; no expensive mutator work; no proof-driven consensus effects.

**Implementation choices still requiring a concrete design:** exact config naming and propagation, conservative/unscored topic policy, quantitative queue/byte/concurrency limits, retry-cache policy, the narrow admitted/completion API finalized with PR2, pending-gossip latency handling, and the smallest adequate network-test fixtures. Recommendations in this report are not evidence these choices were already approved.

### 16.2 Risk register

| Risk | Consequence | Required mitigation |
| --- | --- | --- |
| PR1 binding tip and earlier PR2 base diverge | Prepared and live statement identities disagree or tests validate different targets | Explicit rebaseline/alignment gate and statement-consistency regressions |
| Only parser/enum updated | Subscription filter refuses topic or optional accounting misses it | Whitelist, topic enumeration and actual subscribe tests |
| Old persisted ENR reused | Disabled node advertises support or enabled node appears unaware | Restart/record comparison tests |
| Generic error classifier rejects local failure | Unjustified peer penalty | Typed result routing and negative/local-failure tests |
| Snapshot precheck treated as engine authorization | Duplicate engine work or poisoned bookkeeping | Live serialized paired admission |
| All capacity acquired only after costly preparation | Unbounded pre-admission hashing/BLS/replay | Finite ingress/preparation budgets as well as engine limits |
| Wrong context or retired target accepted late | Wrong-statement storage or resurrection | Cheap current identity and retained-root checks at admission/completion |
| Block-topic scoring copied without justification | Unfair delivery penalties or misleading score expectations | Explicit limited prototype scoring rationale |
| Network-test shortcuts mask behavior | False success from direct routes, duplicate cache or trusted-peer bypass | Controlled topology, fresh identities and relevant unbypassed checks |
| Mock success described as proof validity | False interoperability/security claim | Explicit demonstration label and separate real-proof sign-off |

No proof-specific RPC, new transport, automatic proof-peer maintenance, proof production, consensus-validity policy, persistent proof archive or restart recovery is introduced by this scope. A complete production-hardening claim is not intended.

## 17. Demonstration and handoff checklist

A reviewer or another developer should receive:

- [ ] Exact Grandine, library and spec revisions, plus build toolchain/features.
- [ ] Explicit statement/binding convention and the accepted-payload/state fixture source.
- [ ] A supported proof-type choice and controllable mock engine, never silently enabled as a production verifier.
- [ ] A valid eligible signer and matching fork digest/signing domain.
- [ ] Startup options demonstrating enabled and disabled awareness/subscription behavior.
- [ ] Finite queue/byte/preparation/verification budgets with saturation observations.
- [ ] Publisher → Grandine → observer topology with no direct publisher–observer path.
- [ ] Commands and expected observations for valid, bad-signature, invalid-proof, unavailable-context, duplicate and capacity cases.
- [ ] Evidence of exact stored bare-envelope bytes, not merely a receipt or ACCEPT log.
- [ ] Controlled engine-call counts and late-result/retirement behavior.
- [ ] Correct peer-outcome reporting and post-accept forwarding.
- [ ] Shutdown/expiry limitations and an explicit statement that running verification cancellation is not demonstrated.
- [ ] Passed, failed and skipped checks with test counts and no zero-test pass claims.
- [ ] A clear label: **mock-backed networking/service demonstration; real-proof interoperability not established**.

Library delivery can be signed off for its transport contract before this full checklist is complete. The complete milestone cannot.

## 18. Sources and verification limits

### 18.1 Normative and public project references

- [EIP-8025 proposal][eip].
- [Selected networking specification][spec-p2p], [beacon-chain rules][spec-beacon], [downstream storage][spec-storage] and [proof-engine boundary][spec-engine].
- [Inherited gossip encoding][spec-encoding] and [size requirements][spec-size].
- [Grandine PR1 / #17][pr1], [selected data-model PR / #18][data-model], [current binding adjustment][grandine-binding-change], [preparation source][grandine-prepared] and [engine interface][grandine-engine].
- [Networking-library inspected tree][net-pin], [historical topic patch][net-topic-history] and [whitelist follow-up][net-whitelist-history].
- [Prysm inspected tree][prysm-pin], [validator][prysm-validator], [subscriber][prysm-subscriber], [awareness helper][prysm-awareness], [engine][prysm-engine] and [validator tests][prysm-validator-tests].

All code links are pinned to inspected commits. PR and repository landing pages are live references and can change. Links to existing modules identify architecture/conventions; they are not links to an implemented proof-networking patch. Private planning material is not required to understand this report.

### 18.2 Evidence classification

**Performed:** complete library file inventory; topic/pubsub/ENR/subscription/filter/scoring/metrics/test-harness source tracing; relevant Grandine preparation/task/engine inspection; Prysm networking/validation/recording/error/limit inspection; local revision checks; selected feature-spec committed-diff comparison; historical topic/whitelist diffstat inspection; public PR1 metadata lookup.

**Document checks:** all 18 numbered sections, code fences, reference definitions, final newline and whitespace were checked. All 56 source/reference definitions use public HTTPS URLs; 52 pinned Git object/path targets were checked against the inspected repositories. Six representative public entrypoints returned HTTP 200 to HEAD requests: the three implementation trees, selected networking spec, PR1 page and EIP page. This is not an HTTP/fragment check of every individual link.

**Status input:** the in-progress PR2 readiness summary reflects maintainer-confirmed work status, not a newly executed review of its complete branch or an independently reproduced test run. Its earlier base must be reconciled with the refreshed public PR1 before implementation decisions rely on equivalence.

**Not performed:** Rust or Go test execution, native verification, runtime race reproduction, performance benchmarks, independent vectors, real-proof interoperability, complete workspace CI, all-branch cleanliness audit or a line-by-line audit of unrelated RPC/peer-management code. Go and Bazel were unavailable in the evaluation environment. A prior broad Prysm working-tree check timed out; no full cleanliness claim is made. There is no measured implementation diff because no networking code was changed.

The report's estimate is therefore supported by source structure and historical changes, not measured implementation velocity or demonstrated production capacity. Future acceptance must add executable evidence at the exact integrated revisions.

[eip]: https://eips.ethereum.org/EIPS/eip-8025
[net-repo]: https://github.com/grandinetech/eth2_libp2p
[net-pin]: https://github.com/grandinetech/eth2_libp2p/tree/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a
[net-manifest]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/Cargo.toml
[net-topics]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/types/topics.rs
[net-pubsub]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/types/pubsub.rs
[net-service]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/service/mod.rs
[net-utils]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/service/utils.rs
[net-enr]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/discovery/enr.rs
[net-scoring]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/service/gossipsub_scoring_parameters.rs
[net-metrics]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/metrics.rs
[net-config]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/config.rs
[net-globals]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/types/globals.rs
[net-gossip-cache]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/src/service/gossip_cache.rs
[net-test-common]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/tests/common.rs
[net-test-main]: https://github.com/grandinetech/eth2_libp2p/blob/55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a/tests/main.rs
[net-topic-history]: https://github.com/grandinetech/eth2_libp2p/commit/4f26559f62cdc34e5392bd3a1b3e598c45372db9
[net-whitelist-history]: https://github.com/grandinetech/eth2_libp2p/commit/48db25355ce65ecf403d437790463855b8453a48
[embedded-net]: https://github.com/grandinetech/eth2_libp2p/tree/d03b1685abea2b45b56c65cb7adc5089ca9d5fec
[grandine-pin]: https://github.com/keshavsharma25/grandine/tree/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4
[pr1]: https://github.com/eip8025-grandine/grandine/pull/17
[data-model]: https://github.com/eip8025-grandine/grandine/pull/18
[grandine-binding-change]: https://github.com/keshavsharma25/grandine/commit/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4
[grandine-prepared]: https://github.com/keshavsharma25/grandine/blob/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4/fork_choice_store/src/prepared_execution_proof.rs
[grandine-containers]: https://github.com/keshavsharma25/grandine/blob/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4/types/src/eip8025/containers.rs
[grandine-consts]: https://github.com/keshavsharma25/grandine/blob/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4/types/src/eip8025/consts.rs
[grandine-tasks]: https://github.com/keshavsharma25/grandine/blob/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4/fork_choice_control/src/tasks.rs
[grandine-mutator]: https://github.com/keshavsharma25/grandine/blob/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4/fork_choice_control/src/mutator.rs
[grandine-engine]: https://github.com/keshavsharma25/grandine/blob/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4/proof_engine/src/engine.rs
[grandine-network]: https://github.com/keshavsharma25/grandine/blob/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4/p2p/src/network.rs
[grandine-manifest]: https://github.com/keshavsharma25/grandine/blob/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4/Cargo.toml
[grandine-clippy]: https://github.com/keshavsharma25/grandine/blob/80ed3be0e354ae0c00d87e64ae5b1ba96c662ec4/scripts/ci/clippy.bash
[spec-local]: https://github.com/ethereum/consensus-specs/tree/460fd37c3a58f6baf8a9a285d2adf71c4f818929
[spec-p2p]: https://github.com/ethereum/consensus-specs/blob/12c8b24bc09b12f62688d439ba20ef00b1fdfe1f/specs/_features/eip8025/p2p-interface.md
[spec-gossip]: https://github.com/ethereum/consensus-specs/blob/12c8b24bc09b12f62688d439ba20ef00b1fdfe1f/specs/_features/eip8025/p2p-interface.md#new-execution_proof
[spec-awareness]: https://github.com/ethereum/consensus-specs/blob/12c8b24bc09b12f62688d439ba20ef00b1fdfe1f/specs/_features/eip8025/p2p-interface.md#execution-proof-awareness
[spec-beacon]: https://github.com/ethereum/consensus-specs/blob/12c8b24bc09b12f62688d439ba20ef00b1fdfe1f/specs/_features/eip8025/beacon-chain.md
[spec-storage]: https://github.com/ethereum/consensus-specs/blob/12c8b24bc09b12f62688d439ba20ef00b1fdfe1f/specs/_features/eip8025/fork-choice.md#new-on_execution_proof
[spec-engine]: https://github.com/ethereum/consensus-specs/blob/12c8b24bc09b12f62688d439ba20ef00b1fdfe1f/specs/_features/eip8025/proof-engine.md
[spec-encoding]: https://github.com/ethereum/consensus-specs/blob/12c8b24bc09b12f62688d439ba20ef00b1fdfe1f/specs/phase0/p2p-interface.md#encodings
[spec-size]: https://github.com/ethereum/consensus-specs/blob/12c8b24bc09b12f62688d439ba20ef00b1fdfe1f/specs/phase0/p2p-interface.md#gossipsub-size-limits
[prysm-pin]: https://github.com/OffchainLabs/prysm/tree/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6
[prysm-topics]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/p2p/topics.go
[prysm-mappings]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/p2p/gossip_topic_mappings.go
[prysm-subscription]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/sync/subscriber.go#L389-L405
[prysm-awareness]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/p2p/execution_proof.go
[prysm-discovery]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/p2p/discovery.go#L616-L619
[prysm-encoder]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/p2p/encoder/ssz.go
[prysm-validator]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/sync/validate_execution_proof.go
[prysm-subscriber]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/sync/subscriber_execution_proof.go
[prysm-scoring]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/p2p/gossip_scoring_params.go
[prysm-engine]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/proofengine/engine.go
[prysm-sync-service]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/sync/service.go
[prysm-chain]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/blockchain/execution_proof.go
[prysm-request-root]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/consensus-types/blocks/new_payload_request.go
[prysm-validator-tests]: https://github.com/OffchainLabs/prysm/blob/e99a87cfcaf48d0aed4b32f89b33cb98f357a1e6/beacon-chain/sync/validate_execution_proof_test.go
