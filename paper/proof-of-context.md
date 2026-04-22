# Proof-of-Context: The Missing Verification Layer in Decentralized ML Protocols

**Juan Cruz Maisu**
Buenos Aires, Argentina · juancmaisu@outlook.com · github.com/asastuai

**Version 0.3.1** — **Published: 22 April 2026** (reference corrections patch — see revision note; substantive content identical to v0.3 except where corrected titles/authors/dates affected in-text statements)

---

## Abstract

Decentralized machine-learning protocols have made significant progress on verification primitives such as *proof-of-learning* in distributed training marketplaces (e.g., Gensyn), *zkML proofs paired with TEE attestations* in verifiable-inference networks (e.g., OpenGradient), and *locality-sensitive hashing of inference activations* in production training-inference hybrids (e.g., Prime Intellect's TOPLOC, deployed at 32B scale in INTELLECT-2). A substantial body of work in federated learning also addresses model staleness, asynchronous coordination, and training-verification spoofability. Concurrent and prior work has begun to touch specific facets of the problem we name: Verifiable Dropout (Lee et al., 2025) binds context strings to dropout randomness; TOPLOC (Ong et al., 2025) achieves 0% false-positive and false-negative rates on inference-activation integrity with 258-byte proofs per 32 tokens; PAL\*M (Chantasantitam et al., 2026) composes property attestations across training, inference, and session phases with a challenge-response freshness device under a centralized-provider threat model; CIV (Gupta, 2025) supplies per-token information-flow integrity against prompt injection; Bittensor's Commit-Reveal (Rhodes et al., 2024) uses staleness as cryptoeconomic offense against weight-copiers. These contributions, though, are fragment-level or differently-motivated instances of what this paper argues is a single gap — which we name **proof-of-context** — and which is most productively treated as a *composable verification-primitive layer* sitting on top of proof-of-learning, zkML, TEE attestation, and refereed delegation, binding training state, inference state, and prompt-cache state into one temporally-indexed commitment. The gap is the structural analogue, in ML protocols, of the oracle-freshness problem that accounts for the majority of value-stolen exploits in decentralized finance (2020–2024). We identify four recurring modes of context failure, draw the DeFi analogy explicitly, situate the contribution relative to the existing literature under a decentralized-marketplace threat model, and sketch constraints for a viable construction without yet specifying it. This document establishes the term, the cross-phase unification, and the cryptographic-freshness framing on the public record as of its publication date.

**Keywords:** verifiable computing · decentralized ML · proof-of-learning · zkML · TEE · refereed delegation · locality-sensitive hashing · oracle problem · staleness-aware learning · prompt-cache integrity · adversarial protocols · cryptoeconomic design

---

## 1. Introduction

The emerging class of decentralized ML protocols — training marketplaces, verifiable-inference networks, agent-compute exchanges — have inherited a specific verification posture from their cryptographic roots. The posture is: *prove that the computation happened, prove that it produced the claimed output.* Proof-of-learning does this for training steps. zkML proofs do this for inference. TEE attestations do this for hardware execution. Refereed delegation (Gensyn's Verde [14]) does it for general ML program execution. Locality-sensitive activation hashing (Prime Intellect's TOPLOC [15]) does it for inference integrity at production scale. All five are real, load-bearing, and hard-won.

This paper argues that this posture, while necessary, is *not sufficient*. Verifying that a computation was executed correctly is a different and weaker guarantee than verifying that the computation was executed *in the right context*. We name the latter gap **proof-of-context**.

We are not the first to observe that context — specifically model staleness, asynchronous coordination, and nondeterminism — matters in decentralized ML. A substantial literature in federated learning has studied these problems for roughly half a decade [5–9]. Recent work has also begun binding context strings to individual stochastic operations [13], composing attestations across ML-system phases under a centralized-provider threat model [16], enforcing per-token information-flow integrity [17], and using staleness as a cryptoeconomic offensive against consensus-copying validators [18]. Our contribution is not the discovery of the problem. Our contributions are:

1. **The name** for the unified phenomenon — *proof-of-context*.
2. **The cross-phase unification**: training-state, inference-state, and prompt-cache-state integrity treated as facets of a single primitive, under a decentralized-marketplace threat model (distinguishing our framing from PAL\*M's centralized-provider composition [16]).
3. **The cross-domain analogy** to the oracle-freshness problem in decentralized finance [12].
4. **The extension** of the framing from the training side into the inference and prompt-cache sides where attention-based models introduce new context surfaces — specifically KV-cache staleness, which the existing literature does not treat and which TOPLOC's authors explicitly flag as an uncovered attack surface [15, §6.1].
5. **The framing** of proof-of-context as a *composable verification-primitive layer*, not a monolithic system — admitting Verifiable Dropout, TOPLOC, Verde, Commit-Reveal, and CIV as fragment-level instances.

We believe the unification is worth having because the existing literature's domain-specific vocabulary obscures a pattern the field is going to have to solve coherently at the protocol layer.

The motivating observation is cross-domain. In decentralized finance, the analogous gap — the oracle-freshness problem — has been responsible for the majority of value-stolen exploits across every major AMM, lending protocol, and derivatives platform. Systems that verified "did the swap execute correctly?" failed because they did not verify "was the price you swapped against fresh?" We expect the same pattern to appear in decentralized ML, and we argue below that it is already latent in production systems.

---

## 2. Background: Existing Verification Primitives

We briefly survey five primitives currently deployed in decentralized ML protocols.

**Proof-of-learning.** In protocols such as Gensyn, a worker commits on-chain to intermediate gradient checkpoints at agreed intervals. Verifiers re-run short segments of the training step against the submitted checkpoints and confirm that the gradients match. The guarantee: *the reported gradient was produced by the claimed training operation on the claimed input.* The foundational construction was introduced by Jia et al. [1]. Subsequent work has documented spoofability under certain conditions and sensitivity to hardware nondeterminism [2, 3].

**zkML proofs.** In protocols such as OpenGradient, a worker produces a zero-knowledge proof that a specific model, identified by its weight commitment, was executed on a specific input and produced a specific output. The guarantee: *the reported inference is cryptographically bound to the claimed model + input pair.* The field is surveyed comprehensively in [4].

**TEE attestations.** A Trusted Execution Environment produces a signed attestation that the computation occurred inside the enclave, that the enclave ran the claimed code, and that the output was produced by that code. The guarantee: *the hardware executed the claimed binary honestly.*

**Refereed delegation with deterministic replay.** Gensyn's Verde construction [14] pairs an optimistic refereed-delegation scheme (multiple providers re-execute; disagreement triggers bisection down to a single operation) with a library (RepOps) enforcing bitwise reproducibility across heterogeneous GPU hardware. The guarantee: *the reported ML program execution is bitwise-verifiable against a single honest replica.* The construction explicitly scopes out cache behavior, data-oracle freshness, context-dependent validity, and sampling-algorithm fidelity.

**Inference-activation LSH fingerprinting.** Prime Intellect's TOPLOC [15] hashes the last-layer hidden-state activations of an inference using a top-k polynomial-encoding scheme (258 bytes per 32 tokens, ~1024× reduction), achieving 0% false-positive and 0% false-negative rates against model replacement, prompt modification, and precision downgrade in its empirical regime. It is the closest production primitive to context-binding: a single hash couples prompt, model weights, and precision into one verifiable fingerprint. Its scope is per-inference, post-hoc, and it explicitly does not address speculative decoding, KV-cache compression, subtle fine-tune gradients, or "unstable prompt mining" attack surfaces [15, §6].

Each of these primitives verifies a dimension of computational correctness. Together they cover much of the attack surface a malicious worker might exploit to fabricate work. What they do not cover — individually or in combination as currently deployed — is the question of whether the correct computation, executed on the correct input, was performed under the *right contextual state.*

---

## 3. Related Work

Our contribution sits at the intersection of several existing threads. We organize by overlap severity.

**3.1 Staleness-aware federated and decentralized learning.** A substantial body of work in asynchronous federated learning has studied the effect of model staleness on training quality [5, 6, 7, 8]. Techniques include staleness-weighted gradient aggregation, temperature-controlled weighting, dynamic staleness control, and behavioral staleness metrics [6, 7, 8]. The decentralized-blockchain extension [9] quantified accuracy degradations of up to ~35% attributable to staleness and inconsistency in blockchained federated learning settings. Recent work integrating blockchain with federated learning on edge networks [23] covers the training half of our frame but does not name the primitive, does not address inference or prompt-cache, and does not import the DeFi-oracle analogy. The broader tradition treats staleness as a *coordination and weighting* problem. Our framing shifts the question to whether contextual appropriateness can be *verified* at contribution time, as a primitive, and refused rather than down-weighted.

**3.2 Proof-of-learning robustness.** The foundational construction of Jia et al. [1] was shown to be spoofable by subsequent analyses [2, 10]. Optimistic Verifiable Training [3] addresses nondeterminism specifically. The concern is orthogonal to ours: hardware nondeterminism is about whether the *same* computation can be reproduced bit-exactly, whereas contextual appropriateness is about whether the *right* computation was performed relative to the global state of the training run. Note that [10] (Fang et al., 2022) is a follow-up from the same research group as [1], documenting further spoofability surfaces beyond [2].

**3.3 Verifiable-ML lifecycle taxonomies.** Recent surveys [4, 11] and the SoK of Bruschi et al. [19] partition ML verification into training, testing, inference, and (for FL) aggregation. The SoK's phase-based decomposition — Proof of Committed Data, Proof of Training, Proof of Aggregation — is the scaffold we extend. Proof-of-Context enters this scaffold as the fourth phase covering the post-training inference lifecycle, which the SoK explicitly excludes from its scope [19, §1.1]. A related framework [20] frames training+inference+unlearning as an end-to-end pipeline and enumerates its components; our contribution names the binding primitive that they enumerate. This taxonomy-level framing differs from the primitive-level framings surveyed below.

**3.4 Context-binding in stochastic operations.** Verifiable Dropout [13] is the nearest construction on the binding-mechanism axis. It binds dropout masks to a context payload `(model_id, step, batch_id, verifier-nonce, layer_id)` via a hash-and-sign chain (SHA256 + Ed25519) and proves the result in a zkVM (RISC Zero). This is exactly the shape of context-binding payload we propose to generalize. The scope of [13] is strictly training-side dropout; it never addresses inference, cross-phase, freshness, staleness, oracle semantics, or composable primitives. Its future work lists probabilistic auditing, specialized tensor circuits, and hardware proving — none aligned with our cross-phase framing.

**3.5 Inference integrity via locality-sensitive hashing.** TOPLOC [15] is the production-grade inference-side construction, deployed at 32B-parameter scale in INTELLECT-2. Its empirical performance (0% FPR, 0% FNR across deterministic and non-deterministic GPU settings; 258 bytes per 32 tokens; 0.26 ms/token commit overhead) is state-of-the-art for inference-activation integrity. The construction uses a top-k polynomial-encoding scheme with fixed dual thresholds (for bf16: `Texp=38, Tmean=10, Tmedian=8`). The authors explicitly acknowledge three uncovered surfaces: speculative decoding (by design), FP8 vs. bf16 precision-boundary cases, KV-cache compression (untested), and "unstable prompt mining" as an open adversarial vector [15, §6]. The construction does not carry freshness semantics: each commitment is per-rollout and post-hoc. Our framing positions TOPLOC as an inference-integrity fragment of the proof-of-context primitive and makes the KV-cache and prompt-mining surfaces first-class rather than acknowledged-but-uncovered.

**3.6 Property attestation across phases.** PAL\*M [16] is the closest prior work on the cross-phase-composability axis. It composes "proof of training," "proof of inference," and "proof of session inference" in a single property-attestation framework with a challenge-response "Chal" device for freshness. We must distinguish preemptively and at multiple axes, because the framings are close enough on surface that conflation is likely without explicit separation.

The two primitives differ on four formal axes:

*(i) Subject of attestation.* PAL\*M attests *properties of the model and dataset as static artifacts* — what those artifacts are (architecture, parameter count, training-data commitment, applied operations such as unlearning or fine-tuning). Proof-of-context attests *temporal contextual state of the computation at compute time* — what the surrounding world was at the moment the computation was performed (model version in active use, KV-cache root, data-shard index, optimizer-state root, ambient timestamp).

*(ii) Semantics of freshness.* PAL\*M's freshness mechanism (the "Chal" challenge-response device) is anti-replay: it prevents a provider from caching a single favorable attestation and replaying it to multiple verifiers. Proof-of-context's freshness is temporal-validity: a contribution is *rejected* if its context-commitment is older than a protocol-defined horizon K relative to the current canonical global state.

*(iii) Position in the workflow.* PAL\*M operates as a post-hoc audit on demand (regulator or customer requests; provider attests). Proof-of-context operates as a pre-emptive gate in-flight (if the commitment fails the freshness check, the protocol does not pay). This is the same structural position DeFi's price-staleness check occupies relative to a liquidation: verification before acceptance, not attestation after the fact.

*(iv) Cache-state coverage.* PAL\*M does not model the KV-cache or session-state chain as a verifiable object. Proof-of-context elevates the KV-cache root to a first-class commitment (C4 mode below), covering the inference-runtime provenance channel that PAL\*M's "proof of session inference" touches only as a static property (the session occurred, with what model) rather than as a verifiable state-chain with freshness semantics.

PAL\*M also operates under a *centralized-provider* threat model where a single large-model provider attests properties to an external regulator or customer; proof-of-context operates under a *decentralized-marketplace* threat model where multiple adversarial workers compete for payment from a protocol that cannot rely on any single provider's honesty. The two constructions may share some primitives (signed commitments, Merkle trees, challenge mechanisms) but target distinct failure modes and cannot be substituted for each other. Neither work uses the term "proof-of-context"; PAL\*M does not import the DeFi oracle analogy.

**3.7 Per-token contextual integrity.** Contextual Integrity Verification (CIV) [17] supplies per-token HMAC provenance labels with a trust-lattice attention mask, defending against prompt-injection attacks within a single inference call. CIV's threat model is within-context injection; proof-of-context's threat model is stale-context substitution between calls. CIV is a per-token integrity layer; proof-of-context is a per-session freshness layer. The two are orthogonal and potentially composable.

**3.8 Tensor-native proof-of-inference.** TensorCommitments [21] produces Merkle-style commitments over tensor states during inference with 0.97% overhead on LLaMA2, under the name "proof-of-inference." The name collision with "proof-of-context" is terminological, not semantic: TensorCommitments is an inference-only primitive without cross-phase, cache, or freshness semantics. It is a candidate lower-layer primitive that proof-of-context could compose over.

**3.9 Staleness as cryptoeconomic offense.** Bittensor's Commit-Reveal [18] is the nearest cryptoeconomic precedent for staleness in decentralized ML. Validators commit BLAKE2(weights, nonce) on-chain during a commit window; weights are revealed after a `commit_reveal_period` (default one tempo ≈ 72 minutes on mainnet). Staleness-copying validators depart from Yuma consensus and suffer emergent dividend penalties proportional to the MSE between their (stale) reported weights and the (current) consensus. The mechanism is *producer-side* — it hides producer signals from peer producers to deter consensus-copying. It has no consumer-side freshness semantic: downstream users of the subnet's outputs receive no cryptographic recency guarantee. Proof-of-context occupies the complementary axis: consumer-side freshness gating, not producer-side signal hiding; cryptographic proof of recency, not cryptoeconomic dividend decay.

**3.10 Oracle freshness in DeFi.** The oracle problem in decentralized finance is not typically discussed in the ML literature, but it is directly relevant and underlies the analogy at the center of this paper. The use of time-weighted average prices (TWAPs), multi-source quorums, freshness thresholds, and per-block price deviation checks emerged as cryptoeconomic responses to what was initially treated as a data-quality concern — and is now understood as a verification-layer concern [12]. A recent exploration asks whether AI can *solve* the blockchain oracle problem [25]; our analogy runs in the opposite direction, importing DeFi's hard-won oracle-freshness discipline into ML verification. We argue the ML field is approximately where DeFi was in 2020 on this axis.

**3.11 Adjacent concerns we flag but do not unify.** The RAG-security literature [24] identifies corpus-freshness and provenance-tracking as open problems. Enterprise governance frames such as Context Kubernetes [26] introduce freshness-aware state machines (fresh/stale/expired/conflicted) but in an orchestration rather than cryptographic register. Right to History [27] proposes verifiable agent-execution ledgers. These are related but distinct concerns — none is a cryptographic context-verification primitive under a decentralized threat model.

**Summary.** The literature has surfaced most of the pieces — staleness concerns, spoofability gaps, nondeterminism controls, phase-specific verification primitives, inference-activation integrity, centralized-provider property attestation, within-context integrity, and cryptoeconomic-offensive staleness mechanisms. It has not, to our knowledge, named the unification, proposed the cross-domain DeFi analogy, extended the framing to prompt-cache integrity, or framed the problem as requiring a new composable verification primitive at the decentralized-marketplace protocol layer rather than better scheduling algorithms, stronger centralized attestations, and individually-hardened existing primitives.

---

## 4. The Gap: Computation Correctness vs. Context Appropriateness

Machine-learning training and inference, unlike a token swap, are not memoryless. Every gradient step in a training run depends on the global state of the model at that exact moment — the current weights, the current optimizer state, the current learning-rate schedule, the correct data shard at the correct step. Every inference call depends on the current model version, the current prompt cache (for attention-based systems), the current system prompt, the current set of tool bindings, and so on. The computation is only meaningful in the context in which it was performed.

A worker can produce a mathematically correct gradient on a stale model snapshot. The gradient passes proof-of-learning verification trivially — the math is right. But the gradient is globally useless, or worse: it actively degrades the training run because it pulls the optimizer backward relative to other workers operating on the fresh snapshot. The staleness-aware FL literature addresses this by *weighting* the gradient down in aggregation; we argue the protocol layer should additionally have the option of *refusing to pay* for gradients below a contextual-freshness threshold, as a verification primitive rather than as a post-hoc coefficient.

A worker can produce a valid zkML proof of an inference call on a stale model version. The proof passes verification trivially — the computation on the committed weights is correct. But the client received an answer from a deprecated model. No equivalent of the staleness-aware-FL tradition exists on the inference side yet, which is partly why this framing is needed.

Production inference-integrity systems such as TOPLOC achieve strong guarantees against model replacement, prompt modification, and precision downgrade (0% FPR/FNR in their empirical regime [15]), yet their authors explicitly flag speculative decoding, FP8 precision boundaries, KV-cache compression, and adversarial "unstable prompt mining" as uncovered attack surfaces [15, §6]. All four sit inside the context-verification frame we propose: they are failures of contextual appropriateness rather than computational correctness, and a freshness-indexed context-commitment primitive would address them structurally where per-rollout activation-hashing addresses them probabilistically.

The generalization: **a computation can be correct and still be wrong**, if the context under which it was performed is not the context the protocol intended.

---

## 5. The DeFi Analogy

Decentralized finance has lived with exactly this gap for six years.

Early AMMs verified that swaps executed according to their bonding curves. The swap itself was always correct: given a pool state, the curve produced the deterministically correct output amount. What AMMs did not verify — and what flash-loan attackers exploited repeatedly — was whether the *pool state itself* reflected fresh price information at the moment of the swap. Attackers temporarily manipulated the pool state (via flash loans), swapped against the manipulated state (with computational correctness), and exited. Every swap passed the protocol's verification primitive. Billions of dollars were drained.

Lending protocols tracking collateral value via on-chain oracles suffered the same pattern under the name *stale oracle exploits*. The price feed was occasionally late by one block or one epoch, and attackers used the delay to liquidate healthy positions or borrow against undervalued collateral. The feed was never wrong in the sense of reporting a fabricated number. It was wrong in the sense of reporting a true number *from the wrong moment*.

The DeFi response to this gap, over several years, was the construction of a full secondary verification layer around *context verification* — freshness, deviation thresholds, time-weighted averages, multiple-source quorums, circuit breakers [12]. The primary verification of computational correctness stayed where it was. But on top of it, the industry built a context layer that became load-bearing for safety.

**Claim.** The decentralized-ML protocol stack is, in 2026, roughly where DeFi was in 2020 on this axis. The primary verification layer (PoL, zkML, TEE, refereed delegation, inference-activation LSH) is hardening quickly; its robustness concerns are well-studied [1–3, 10, 14, 15]. The staleness-weighting literature [5–9] addresses context as a scheduling concern. Centralized-provider property attestation [16] composes phases but under a different threat model. A context-verification primitive at the decentralized-protocol level, composing across training, inference, and prompt-cache, with cryptographic freshness semantics imported from DeFi oracle design, does not yet exist in the literature. The gap is being paid for silently — in degraded training runs, in deprecated-model inferences, in rewards flowing to workers who computed correctly but contributed nothing useful.

---

## 6. Modes of Context Failure

Four recurring modes, all of which pass existing verification primitives while violating the protocol's practical intent:

**C1. Stale model snapshot.** A training worker computes gradients against a model version that is multiple global updates behind. Each individual gradient is correct relative to its local starting point. Aggregated into the global optimizer, the gradient pushes the model toward a previous state [5, 9]. An inference worker serves a deprecated model version. Each output is correct for the weights used. The client receives outdated intelligence.

**C2. Data-shard desynchronization.** A training worker receives a data shard that was intended for a different training step or a different curriculum phase. The gradient is mathematically correct on the data received. It is globally harmful because it represents the wrong curriculum signal at the wrong moment of training. (Partial overlap with Verifiable Dropout's threat model [13] at the per-step level, but the cross-phase and freshness dimensions are outside its scope.)

**C3. Optimizer-state drift.** A worker computes gradients with an optimizer state (momentum terms, second moments, adaptive learning rates) that is out of sync with the global optimizer [8]. The gradient math is correct. The global optimizer update it produces is not — the momentum component is wrong relative to the global trajectory.

**C4. Prompt-cache / context-window staleness.** In inference for attention-based models, a worker operates against a stale prompt cache or a stale system prompt, or substitutes a compressed or modified KV-cache. The inference call is computationally correct against the cached state. The user receives an output conditioned on outdated or substituted context. This mode has no direct analogue in the federated-learning staleness literature because that literature predates modern attention-based inference. TOPLOC's authors explicitly acknowledge KV-cache compression and "unstable prompt mining" as uncovered surfaces [15, §6]; CIV [17] addresses within-context injection but not cross-call cache substitution. C4 is where the proof-of-context framing extends furthest beyond any existing primitive.

All four are invisible to PoL, zkML, TEE, and refereed delegation in their current forms. TOPLOC partially covers C1 (inference side) and C4 (prompt modification but not KV-cache substitution) as a post-hoc per-rollout hash. Verifiable Dropout partially covers an adjacent aspect of C2 for single stochastic operations. PAL\*M covers analogous concerns for centralized providers. None covers all four modes as a composable freshness-indexed primitive under a decentralized-marketplace threat model.

---

## 7. Prior Art from the Author's Portfolio

Two artifacts from the author's production work (2025–2026) anticipated components of this problem from the agent-infrastructure side, prior to formalizing the proof-of-context framing:

**TrustLayer** ([github.com/asastuai/TrustLayer](https://github.com/asastuai/TrustLayer)) — An agent-reputation infrastructure layer on Base L2 combining skill audit, test harness, uptime monitoring, and payment escrow into a single x402-native API. TrustLayer's reputation score is a rough historical sketch of *contextually-appropriate behavior over time* — the dimension that proof-of-context systems would formalize into a real-time verification primitive rather than an after-the-fact scoring mechanism.

**BaseOracle** ([github.com/asastuai/BaseOracle](https://github.com/asastuai/BaseOracle)) — A pay-per-query data oracle for autonomous AI agents on Base, priced via x402 micropayments. BaseOracle enforces contextual freshness on a per-query basis at the data layer and embeds it in the payment primitive; the same structural insight — *freshness must be verified at pay-time, not discovered retroactively* — is the insight proof-of-context generalizes to the compute layer.

Neither artifact is a solution to the proof-of-context problem as formulated here. Both are prior evidence that the author was circling the same structural gap from an adjacent angle before naming it.

---

## 8. A Research Program

We do not in this paper propose a cryptographic construction for proof-of-context. That construction is the central open research question the framing invites. We do, however, sketch the constraints any viable construction must satisfy, informed by the Phase 2 literature survey.

1. **Freshness is verifiable per-computation, not reputational.** The protocol must be able to reject a specific gradient / inference / contribution for contextual staleness without relying on long-horizon reputation scores. (Cf. the DeFi evolution away from "trusted reporter" oracle models and toward per-query freshness proofs.)

2. **Context commitments must be cheap.** If committing to the contextual state of a training step costs more than the training step itself, the economics fail. The construction likely requires Merkle commitments, hash chains, or similar amortizable structures on top of existing PoL / zkML / TEE / LSH primitives, not parallel to them. TOPLOC's 258 bytes per 32 tokens sets a useful cost benchmark for the inference side [15].

3. **Cross-worker consistency is required.** Training protocols cannot verify contextual appropriateness by asking a single worker to report its own context. The construction must cross-reference context claims across a worker set or against a canonical global state published on-chain.

4. **Graceful degradation.** The construction must handle the boundary case where a worker's context is slightly stale but not exploitably so. Binary pass/fail schemes will reject too many honest workers; continuous scoring schemes collapse into reputation in disguise. A *contextual tolerance bound*, expressed in protocol-agnostic terms, is likely a required primitive, and likely has something to learn from the tolerance mechanisms that staleness-aware FL [5–7] uses algorithmically and from the dual-threshold scheme TOPLOC uses for GPU non-determinism [15, §5.2], both at a different layer.

5. **Compatibility with existing verification primitives.** Proof-of-context is a layer *on top of* PoL / zkML / TEE / refereed delegation / LSH fingerprinting, not a replacement. Concretely:
   - It extends the context-payload shape used by Verifiable Dropout [13] from seed-binding for a single stochastic operation to first-class commitment over per-computation state;
   - It composes with the activation-integrity primitive of TOPLOC [15] from per-rollout hashing to temporally-indexed commitment with explicit freshness semantics;
   - It admits the refereed-delegation primitive of Verde [14] to accept freshness-bound challenges alongside math-correctness challenges;
   - It inverts the staleness treatment of Bittensor Commit-Reveal [18] from producer-side hiding to consumer-side freshness gating, borrowing Drand-beacon-based time-locked encryption as a construction primitive where useful.

6. **Freshness semantics borrowed explicitly from DeFi.** The construction should import the cryptoeconomic disciplines that DeFi hardened over six years: heartbeat (every PoC commitment carries a timestamp no older than a protocol-defined horizon), deviation threshold (a commitment may be valid even if stale provided cache-state-distance from canonical is below a bound), multi-source quorum (critical context such as model version must be attested by a committee, not a single worker), and circuit breakers (sudden deviation in the freshness signal triggers protocol-level halting).

The author is actively developing, under the framing above, an initial construction of proof-of-context for a specific class of decentralized-ML protocols. The construction and benchmark results will be published as a companion paper on the basis of this position paper.

---

## 9. Conclusion

The decentralized-ML protocol stack has absorbed the lesson that computations must be verifiably correct. The federated-learning literature has absorbed the lesson that asynchronous staleness must be *scheduled and weighted*. Production inference networks have absorbed the lesson that activation fingerprints can detect model and prompt modification at scale [15]. Centralized-provider property-attestation frameworks have absorbed the lesson that phases can be composed under challenge-response freshness [16]. What has not yet happened, to our knowledge, is the unification at the decentralized-marketplace layer: treating context-appropriateness as a single composable verification-primitive gap that spans training, inference, and prompt-cache; that has a clear structural precedent in the DeFi oracle-freshness problem; and that gates contribution acceptance via cryptographic temporal-validity rather than via post-hoc weighting, cryptoeconomic dividend decay, or centralized-provider attestation.

Our nine-protocol survey (Ritual, Atoma, Nillion, io.net, Aethir, Gensyn/Verde, OpenGradient, Prime Intellect, Inference Labs, with notes on Supra and Hyperbolic) confirms that no production decentralized-ML protocol currently binds `(prompt, KV-cache state, model version, timestamp)` into a single freshness-bound verifiable object that downstream contracts can check the way a DeFi contract checks a Chainlink round. The gap is latent, named for the first time here.

We publish this paper to (a) establish the term *proof-of-context*, the cross-phase unification, the cross-domain DeFi analogy, and the extension to prompt-cache integrity on the academic record as of 22 April 2026, (b) situate the contribution honestly against the substantial existing work in staleness-aware federated learning, proof-of-learning robustness, centralized-provider property attestation, inference-activation LSH, and cryptoeconomic-offensive staleness, and (c) invite protocol teams, independent researchers, and cryptographic-primitive designers to begin treating proof-of-context as a first-class composable gap in the stack rather than a collection of phase-specific or differently-motivated concerns.

The analogy is clean: 2026 decentralized-ML is 2020 DeFi. The next three years of work on the context-verification layer will determine whether the field pays for its stale-oracle equivalents the same way DeFi did.

---

## Acknowledgements

This paper emerged from a 46-day human-AI research collaboration documented in the public narrative *Opus* ([asastuai.github.io/opus](https://asastuai.github.io/opus/)), conducted with Claude (Anthropic) as co-author. The framing crystallized during preparation materials for a 2026 research-engineering application; the term *proof-of-context* was surfaced in collaborative dialogue and subsequently formalized for this paper. The v0.2 revision (21 April 2026) tightened claims and added the Related Work section after a first-pass literature review surfaced the substantial body of existing staleness-aware federated learning work. The v0.3 revision (22 April 2026) incorporated a 48-hour Phase 2 deep-dive covering Verifiable Dropout, TOPLOC, SoK: Verifiable FL, and Bittensor's Weight-Copier paper in full, an obsessive prior-art scan across arXiv / OpenReview / IACR ePrint / conference archives, and a nine-protocol technical survey of production verification stacks — the findings of which are documented in full in `RESEARCH-LANDSCAPE-v2.md` in the paper's git repository. The author is grateful for the intellectual partnership that produced both the naming and the successive corrections.

---

## References

[1] Jia, H., Yaghini, M., Choquette-Choo, C. A., Dullerud, N., Thudi, A., Chandrasekaran, V., & Papernot, N. (2021). *Proof-of-Learning: Definitions and Practice.* IEEE Symposium on Security and Privacy. arXiv:2103.05633.

[2] Zhang, R., Liu, J., Ding, Y., Wu, Z., Wang, Q., & Ren, K. (2022). *"Adversarial Examples" for Proof-of-Learning.* IEEE Symposium on Security and Privacy 2022. arXiv:2108.09454.

[3] Srivastava, M., Arora, S., & Boneh, D. (2024). *Optimistic Verifiable Training by Controlling Hardware Nondeterminism.* arXiv:2403.09603.

[4] Peng, Z., Zhao, J., Wang, R., Liao, X., Lin, J., Liu, Y., Cao, J., Shi, Y., Yang, L., & Zhang, M. (2025). *A Survey of Zero-Knowledge Proof Based Verifiable Machine Learning.* arXiv:2502.18535.

[5] Zhang, W., Gupta, S., Lian, X., & Liu, J. (2016). *Staleness-aware Async-SGD for Distributed Deep Learning.* IJCAI 2016, pp. 2350–2356. arXiv:1511.05950.

[6] Chen, M., Mao, B., & Ma, T. (2021). *FedSA: A staleness-aware asynchronous Federated Learning algorithm with non-IID data.* Future Generation Computer Systems, vol. 120.

[7] Ma, L., Liu, Y., Jia, S., Zhou, J., Hu, Y., & Xie, X. (2024). *Dynamic Staleness Control for Asynchronous Federated Learning in Decentralized Topology.* WASA 2024, Springer LNCS vol. 14998. DOI 10.1007/978-3-031-71467-2_9.

[8] Lu, C., Sun, Y., Yang, Z., Chen, J., Yin, D., & Zhu, J. (2026). *FedPSA: Modeling Behavioral Staleness in Asynchronous Federated Learning.* arXiv:2602.15337.

[9] Wilhelmi, F., Afraz, N., Guerra, E., & Dini, P. (2023). *The Implications of Decentralization in Blockchained Federated Learning: Evaluating the Impact of Model Staleness and Inconsistencies.* arXiv:2310.07471.

[10] Fang, C., Jia, H., Thudi, A., Yaghini, M., Choquette-Choo, C. A., Dullerud, N., Chandrasekaran, V., & Papernot, N. (2022). *Proof-of-Learning is Currently More Broken Than You Think.* arXiv:2208.03567.

[11] Xing, Z., Zhang, Z., Zhang, et al. (2023). *Zero-Knowledge Proof-based Verifiable Decentralized Machine Learning in Communication Network: A Comprehensive Survey.* arXiv:2310.14848.

[12] Adams, H., Zinsmeister, N., Salem, M., Keefer, R., & Robinson, D. (2021). *Uniswap v3 Core.* Uniswap whitepaper, March 2021. Time-weighted average price oracle construction and related DeFi literature on context verification.

[13] Lee, H., Lee, J., Jin, S., & Ko, M. (2025). *Verifiable Dropout: Turning Randomness into a Verifiable Claim.* arXiv:2512.22526.

[14] Arun, A., St. Arnaud, T., Titov, A., Wilcox, B., Kolobaric, V., Brinkmann, M., Ersoy, O., Fielding, B., & Bonneau, J. (Gensyn). (2025). *Verde: Verification via Refereed Delegation for Machine Learning Programs.* arXiv:2502.19405.

[15] Ong, J. M., Di Ferrante, M., Pazdera, A., Garner, R., Jaghouar, S., Basra, M., Ryabinin, M., & Hagemann, J. (Prime Intellect). (2025). *TOPLOC: A Locality Sensitive Hashing Scheme for Trustless Verifiable Inference.* arXiv:2501.16007.

[16] Chantasantitam, T., Caulfield, T., Duddu, V., Gunn, L., & Asokan, N. (2026). *PAL\*M: Property Attestation for Large Generative Models.* arXiv:2601.16199.

[17] Gupta, A. (2025). *Can AI Keep a Secret? Contextual Integrity Verification: A Provable Security Architecture for LLMs.* arXiv:2508.09288.

[18] Rhodes, S., Rao, A. (Opentensor Foundation). (2024). *Weight Copying in Bittensor: A Working Paper.* [docs.learnbittensor.org/papers/BT_Weight_Copier-29May2024.pdf](https://docs.learnbittensor.org/papers/BT_Weight_Copier-29May2024.pdf).

[19] Bruschi, P., Esposito, F., Gagliardoni, T., & Rizzini, F. (2025). *SoK: Verifiable Federated Learning.* IACR ePrint 2025/2296.

[20] Balan, O., Learney, R., & Wood, G. (2025). *A Framework for Cryptographic Verifiability of End-to-End AI Pipelines.* arXiv:2503.22573.

[21] Baser, O., Sadeghi, A., Wang, J., Alves, M., Kazemian, H., Kang, S., Chinchali, S., & Vishwanath, S. (2026). *TensorCommitments: A Lightweight Verifiable Inference for Language Models.* arXiv:2602.12630. Reports 0.97% overhead on LLaMA2.

[22] Prime Intellect. (2025). *INTELLECT-2: A Reasoning Model Trained Through Globally Decentralized Reinforcement Learning.* arXiv:2505.07291. See also: *Prime Collective Communications Library — Technical Report.* arXiv:2505.14065.

[23] Chen, H., Zhou, R., Chan, Y.-H., Jiang, Z., Chen, X., & Ngai, E. C. H. (2025). *LiteChain: A Lightweight Blockchain for Verifiable and Scalable Federated Learning in Massive Edge Networks.* arXiv:2503.04140.

[24] Xu, Z., Zhang, Y., Ge, X., Li, J., Hu, Z., Zhang, W., Li, Z., & Chen, K. (2026). *Securing Retrieval-Augmented Generation: A Taxonomy of Attacks, Defenses, and Future Directions.* arXiv:2604.08304.

[25] Caldarelli, G. (2025). *Can Artificial Intelligence solve the blockchain oracle problem? Unpacking the Challenges and Possibilities.* arXiv:2507.02125.

[26] Mouzouni, C. (2026). *Context Kubernetes: Declarative Orchestration of Enterprise Knowledge for Agentic AI Systems.* arXiv:2604.11623.

[27] Zhang, J. (2026). *Right to History: A Sovereignty Kernel for Verifiable AI Agent Execution.* arXiv:2602.20214.

[28] Supra Research. (2025). *Threshold AI Oracles: Verified AI for Event-Driven Web3.* [supra.com/documents/Threshold_AI_Oracles_Supra.pdf](https://supra.com/documents/Threshold_AI_Oracles_Supra.pdf), 22 May 2025.

[29] OpenGradient Foundation. (2026). *OpenGradient: Decentralized Infrastructure for Verifiable AI Execution.* Whitepaper, March 2026. [opengradient.foundation/whitepaper](https://opengradient.foundation/whitepaper).

[30] Author's prior work: [github.com/asastuai](https://github.com/asastuai) — Hermetic Computing, intent-cipher, SUR Protocol, LiquidClaw Finance, PayClaw, BaseOracle, TrustLayer.

---

## License and Citation

This paper is released under CC BY 4.0. To cite:

> Maisu, J. C. (2026). *Proof-of-Context: The Missing Verification Layer in Decentralized ML Protocols* (Version 0.3.1) [Position paper]. github.com/asastuai/proof-of-context, 22 April 2026.

---

## Revision note

**v0.3.1 (22 April 2026, reference corrections patch):** A post-publication audit against primary sources (arxiv.org, journal pages, publisher PDFs) revealed systematic errors in the v0.3 reference list, introduced during Phase 2 literature survey when titles reported by background-research agents were propagated without independent verification. Specifically: (a) reference to "ChainDrive-FL-VRA" in Nature Scientific Reports — this paper does not exist, the citation has been removed and in-text mention of "ChainDrive-FL-VRA" in §3.1 replaced with accurate reference to LiteChain [23]; (b) reference to Fang & Gong (2021) "On the Robustness of Proof-of-Learning" — no such paper exists; replaced with the real follow-up paper by the same research lineage as [1], namely Fang, Jia, Thudi, et al. (2022) "Proof-of-Learning is Currently More Broken Than You Think" (arXiv:2208.03567); (c) fifteen further references had paraphrased titles (a known LLM failure mode termed *title-paraphrasing drift* — the arXiv IDs and author lists resolve to real papers, but titles had been rewritten by upstream research agents with topic-adjacent buzzwords); these have been corrected to the verbatim titles as recorded on arxiv.org. Author lists have also been corrected where wrong authors had been cited (refs [4], [6], [11], [12] updated). Venue claim on TOPLOC [15] corrected from "ICML 2025" (unsupported) to arXiv-only. Reference [29] (Supra) date corrected from 2026 to 2025. No substantive change to arguments, claims, or conclusions of the paper — all primary findings hold on the basis of correctly-cited sources. This correction is published as a full patch rather than a silent edit because academic integrity requires that errors in the published record be acknowledged with the same timestamp visibility as the original record.

**v0.3 (22 April 2026):** Incorporated Phase 2 literature survey findings. Abstract revised to acknowledge closest-adjacency prior art explicitly (Verifiable Dropout, TOPLOC, PAL\*M, CIV, Bittensor Commit-Reveal) and to sharpen the novelty claims as (a) naming, (b) cross-phase unification under a decentralized-marketplace threat model, (c) cross-domain DeFi analogy, (d) extension to prompt-cache integrity, and (e) composable-primitive framing. §2 Background expanded from three primitives to five (added refereed delegation / Verde and inference-activation LSH / TOPLOC). §3 Related Work restructured with eleven subsections. §3.6 in particular distinguishes proof-of-context from PAL\*M along four formal axes (subject of attestation, semantics of freshness, position in the workflow, cache-state coverage). §4 Gap expanded with TOPLOC's explicit uncovered-attack-surface acknowledgements. §6 Modes revised to note partial coverage by existing primitives per mode. §8 Research Program constraint (5) expanded with a composition map over Verifiable Dropout, TOPLOC, Verde, and Bittensor, and constraint (6) added importing DeFi freshness semantics (heartbeat, deviation, quorum, circuit breakers). §9 Conclusion strengthened with a nine-protocol survey summary. References expanded from 13 to 31. The full Phase 2 research landscape is documented in `RESEARCH-LANDSCAPE-v2.md` in the paper's git repository. v0.2 and v0.1 commits remain in git history for priority purposes.

**v0.2 (21 April 2026):** Added Related Work section and tightened claims after first-pass literature review. No changes to four modes, DeFi analogy, prior-art, or research program.

**v0.1 (21 April 2026):** Initial publication establishing the term and the unified framing on the public record.
