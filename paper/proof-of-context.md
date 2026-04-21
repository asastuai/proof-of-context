# Proof-of-Context: The Missing Verification Layer in Decentralized ML Protocols

**Juan Cruz Maisu**
Buenos Aires, Argentina · juancmaisu@outlook.com · github.com/asastuai

**Version 0.2** — **Published: 21 April 2026** (revised from v0.1 same day with expanded Related Work and tightened claims)

---

## Abstract

Decentralized machine-learning protocols have made significant progress on verification primitives such as *proof-of-learning* in distributed training marketplaces (e.g., Gensyn) and *zkML proofs paired with TEE attestations* in verifiable-inference networks (e.g., OpenGradient). A substantial body of work in federated learning also addresses **model staleness**, asynchronous coordination, and training-verification spoofability. These contributions, though, are largely treated as domain-specific algorithmic concerns: staleness is addressed through scheduling and weighting heuristics, proof-of-learning is examined for robustness against forgery, and nondeterminism is controlled via trusted-execution infrastructure. This position paper argues that these fragments are facets of a single gap — which we name **proof-of-context** — and that the gap is most productively treated as a *verification-primitive layer*, not only as scheduling correctness. The gap is the structural analogue, in ML protocols, of the oracle-freshness problem that accounts for the majority of value-stolen exploits in decentralized finance (2020–2024). We identify four recurring modes of context failure, draw the DeFi analogy explicitly, situate the contribution relative to the existing literature, and sketch constraints for a viable construction without yet specifying it. This document establishes the term and the unified framing on the public record as of its publication date.

**Keywords:** verifiable computing · decentralized ML · proof-of-learning · zkML · oracle problem · staleness-aware learning · adversarial protocols · cryptoeconomic design

---

## 1. Introduction

The emerging class of decentralized ML protocols — training marketplaces, verifiable-inference networks, agent-compute exchanges — have inherited a specific verification posture from their cryptographic roots. The posture is: *prove that the computation happened, prove that it produced the claimed output.* Proof-of-learning does this for training steps. zkML proofs do this for inference. TEE attestations do this for hardware execution. All three are real, load-bearing, and hard-won.

This paper argues that this posture, while necessary, is *not sufficient*. Verifying that a computation was executed correctly is a different and weaker guarantee than verifying that the computation was executed *in the right context*. We name the latter gap **proof-of-context**.

We are not the first to observe that context — specifically model staleness, asynchronous coordination, and nondeterminism — matters in decentralized ML. A substantial literature in federated learning has studied these problems for roughly half a decade. Our contribution is not the discovery of the problem. Our contribution is (a) **the name** for the unified phenomenon, (b) **the framing** of it as a verification-primitive layer rather than a scheduling concern, (c) **the cross-domain analogy** to the oracle-freshness problem in decentralized finance, and (d) **the extension** of the framing from the training side into the inference and prompt-cache sides where attention-based models introduce new context surfaces. We believe the unification is worth having because the existing literature's domain-specific vocabulary obscures a pattern the field is going to have to solve coherently at the protocol layer.

The motivating observation is cross-domain. In decentralized finance, the analogous gap — the oracle-freshness problem — has been responsible for the majority of value-stolen exploits across every major AMM, lending protocol, and derivatives platform. Systems that verified "did the swap execute correctly?" failed because they did not verify "was the price you swapped against fresh?" We expect the same pattern to appear in decentralized ML, and we argue below that it is already latent in production systems.

---

## 2. Background: Existing Verification Primitives

We restrict attention to three primitives currently deployed in decentralized ML protocols.

**Proof-of-learning.** In protocols such as Gensyn, a worker commits on-chain to intermediate gradient checkpoints at agreed intervals. Verifiers re-run short segments of the training step against the submitted checkpoints and confirm that the gradients match. The guarantee: *the reported gradient was produced by the claimed training operation on the claimed input.* The foundational construction was introduced by Jia et al. (2021) [1]. Subsequent work has documented that this construction is spoofable under certain conditions and sensitive to hardware nondeterminism [2, 3] — a robustness concern distinct from the contextual concern we name below.

**zkML proofs.** In protocols such as OpenGradient, a worker produces a zero-knowledge proof that a specific model, identified by its weight commitment, was executed on a specific input and produced a specific output. The guarantee: *the reported inference is cryptographically bound to the claimed model + input pair.* The field is comprehensively surveyed in [4].

**TEE attestations.** A Trusted Execution Environment produces a signed attestation that the computation occurred inside the enclave, that the enclave ran the claimed code, and that the output was produced by that code. The guarantee: *the hardware executed the claimed binary honestly.*

Each of these primitives verifies a dimension of computational correctness. Together they cover much of the attack surface a malicious worker might exploit to fabricate work. What they do not cover is the question of whether the correct computation, executed on the correct input, was nonetheless performed under the *wrong contextual state.*

---

## 3. Related Work

Our contribution sits at the intersection of three existing threads of research, each of which addresses a piece of the problem we unify.

**Staleness-aware federated and decentralized learning.** A substantial body of work in asynchronous federated learning has studied the effect of model staleness on training quality [5, 6, 7, 8]. Techniques include staleness-weighted gradient aggregation, temperature-controlled weighting, dynamic staleness control, and behavioral staleness metrics [6, 7, 8]. This literature treats staleness as a *coordination and weighting* problem — the goal is to allow asynchronous workers to contribute productively despite being out of sync. The decentralized-blockchain extension has also been measured empirically [9], which quantified accuracy degradations of up to ~35% attributable to staleness and inconsistency in blockchained federated learning settings. The angle is algorithmic and post-hoc: how do we *absorb* context failure into the aggregation rule? Our framing shifts the question to whether contextual appropriateness can be *verified* at contribution time, as a primitive, and refused rather than down-weighted.

**Proof-of-learning and its robustness.** The foundational proof-of-learning construction of Jia et al. (2021) [1] was shown to be spoofable by subsequent analyses [2, 10]. The follow-up work "Optimistic Verifiable Training by Controlling Hardware Nondeterminism" [3] addresses one specific robustness concern (nondeterminism in training) and proposes a framework for making proof-of-learning deterministic enough to verify. The concern is orthogonal to ours: hardware nondeterminism is about whether the *same* computation can be reproduced bit-exactly, whereas contextual appropriateness is about whether the *right* computation was performed relative to the global state of the training run.

**Verifiable training and inference as a composite lifecycle.** Recent surveys [4, 11] partition ML verification into verifiable training, verifiable testing, and verifiable inference, each with its own zero-knowledge and cryptographic primitives. This partition helps organize the field but obscures the cross-phase structural pattern we name: namely, that each phase has its own context-appropriateness gap (gradient freshness in training, model-version freshness in inference, prompt-cache freshness in attention-based inference) and that these share a common structural shape that would benefit from a common verification primitive.

**Oracle freshness in DeFi.** The oracle problem in decentralized finance is not typically discussed in the ML literature, but it is directly relevant and underlies the analogy at the center of this paper. The use of time-weighted average prices (TWAPs), multi-source quorums, freshness thresholds, and per-block price deviation checks emerged as cryptoeconomic responses to what was initially treated as a data-quality concern — and is now understood as a verification-layer concern [12]. We argue the ML field is approximately where DeFi was in 2020 on this axis.

**Summary.** The literature has surfaced the pieces — staleness concerns, spoofability gaps, nondeterminism controls, and phase-specific verification primitives. It has not, to our knowledge, named the unification, proposed the cross-domain analogy, or framed the problem as requiring a new verification primitive rather than better scheduling algorithms and individually-hardened existing primitives.

---

## 4. The Gap: Computation Correctness vs. Context Appropriateness

Machine-learning training and inference, unlike a token swap, are not memoryless. Every gradient step in a training run depends on the global state of the model at that exact moment — the current weights, the current optimizer state, the current learning-rate schedule, the correct data shard at the correct step. Every inference call depends on the current model version, the current prompt cache (for attention-based systems), the current system prompt, the current set of tool bindings, and so on. The computation is only meaningful in the context in which it was performed.

A worker can produce a mathematically correct gradient on a stale model snapshot. The gradient passes proof-of-learning verification trivially — the math is right. But the gradient is globally useless, or worse: it actively degrades the training run because it pulls the optimizer backward relative to other workers operating on the fresh snapshot. The primitive passes, the protocol pays, the model gets worse. The staleness-aware FL literature addresses this by *weighting* the gradient down in aggregation; we argue the protocol layer should additionally have the option of *refusing to pay* for gradients below a contextual-freshness threshold, as a verification primitive rather than as a post-hoc coefficient.

A worker can produce a valid zkML proof of an inference call on a stale model version. The proof passes verification trivially — the computation on the committed weights is correct. But the client received an answer from a deprecated model. The primitive passes, the protocol pays, the user is silently served degraded intelligence. No equivalent of the staleness-aware-FL tradition exists on the inference side yet, which is partly why this framing is needed.

The generalization: **a computation can be correct and still be wrong**, if the context under which it was performed is not the context the protocol intended.

---

## 5. The DeFi Analogy

Decentralized finance has lived with exactly this gap for six years.

Early AMMs verified that swaps executed according to their bonding curves. The swap itself was always correct: given a pool state, the curve produced the deterministically correct output amount. What AMMs did not verify — and what flash-loan attackers exploited repeatedly — was whether the *pool state itself* reflected fresh price information at the moment of the swap. Attackers temporarily manipulated the pool state (via flash loans), swapped against the manipulated state (with computational correctness), and exited. Every swap passed the protocol's verification primitive. Billions of dollars were drained.

Lending protocols tracking collateral value via on-chain oracles suffered the same pattern under the name *stale oracle exploits*. The price feed was occasionally late by one block or one epoch, and attackers used the delay to liquidate healthy positions or borrow against undervalued collateral. The feed was never wrong in the sense of reporting a fabricated number. It was wrong in the sense of reporting a true number *from the wrong moment*.

The DeFi response to this gap, over several years, was the construction of a full secondary verification layer around *context verification* — freshness, deviation thresholds, time-weighted averages, multiple-source quorums, circuit breakers [12]. The primary verification of computational correctness stayed where it was. But on top of it, the industry built a context layer that became load-bearing for safety.

**Claim.** The decentralized-ML protocol stack is, in 2026, roughly where DeFi was in 2020 on this axis. The primary verification layer (PoL, zkML, TEE) is hardening quickly; its robustness concerns are well-studied [1, 2, 3, 10]. The staleness-weighting literature [5, 6, 7, 8] addresses context as a scheduling concern. What is not yet in place is a context verification layer at the protocol primitive level. The gap is being paid for silently — in degraded training runs, in deprecated-model inferences, in rewards flowing to workers who computed correctly but contributed nothing useful.

---

## 6. Modes of Context Failure

Four recurring modes, all of which pass existing verification primitives while violating the protocol's practical intent:

**C1. Stale model snapshot.** A training worker computes gradients against a model version that is multiple global updates behind. Each individual gradient is correct relative to its local starting point. Aggregated into the global optimizer, the gradient pushes the model toward a previous state [5, 9]. An inference worker serves a deprecated model version. Each output is correct for the weights used. The client receives outdated intelligence.

**C2. Data-shard desynchronization.** A training worker receives a data shard that was intended for a different training step or a different curriculum phase. The gradient is mathematically correct on the data received. It is globally harmful because it represents the wrong curriculum signal at the wrong moment of training.

**C3. Optimizer-state drift.** A worker computes gradients with an optimizer state (momentum terms, second moments, adaptive learning rates) that is out of sync with the global optimizer [8]. The gradient math is correct. The global optimizer update it produces is not — the momentum component is wrong relative to the global trajectory.

**C4. Prompt-cache / context-window staleness.** In inference for attention-based models, a worker operates against a stale prompt cache or a stale system prompt. The inference call is computationally correct against the cached state. The user receives an output conditioned on outdated context. This mode has no direct analogue in the federated-learning staleness literature because that literature predates modern attention-based inference; we surface it here as a distinct mode that the framing needs to cover.

All four are invisible to PoL, zkML, and TEE in their current forms. The first three overlap with the staleness-aware FL literature as *measurement and weighting* problems; we argue they are also *verification-primitive* problems. The fourth is new.

---

## 7. Prior Art from the Author's Portfolio

Two artifacts from the author's production work (2025–2026) anticipated components of this problem from the agent-infrastructure side, prior to formalizing the proof-of-context framing:

**TrustLayer** ([github.com/asastuai/TrustLayer](https://github.com/asastuai/TrustLayer)) — An agent-reputation infrastructure layer on Base L2 combining skill audit, test harness, uptime monitoring, and payment escrow into a single x402-native API. TrustLayer's reputation score is a rough historical sketch of *contextually-appropriate behavior over time* — the dimension that proof-of-context systems would formalize into a real-time verification primitive rather than an after-the-fact scoring mechanism.

**BaseOracle** ([github.com/asastuai/BaseOracle](https://github.com/asastuai/BaseOracle)) — A pay-per-query data oracle for autonomous AI agents on Base, priced via x402 micropayments. BaseOracle enforces contextual freshness on a per-query basis at the data layer and embeds it in the payment primitive; the same structural insight — *freshness must be verified at pay-time, not discovered retroactively* — is the insight proof-of-context generalizes to the compute layer.

Neither artifact is a solution to the proof-of-context problem as formulated here. Both are prior evidence that the author was circling the same structural gap from an adjacent angle before naming it.

---

## 8. A Research Program

We do not in this paper propose a cryptographic construction for proof-of-context. That construction is the central open research question the framing invites. We do, however, sketch the constraints any viable construction must satisfy:

1. **Freshness is verifiable per-computation, not reputational.** The protocol must be able to reject a specific gradient / inference / contribution for contextual staleness without relying on long-horizon reputation scores. (Cf. the DeFi evolution away from "trusted reporter" oracle models and toward per-query freshness proofs.)

2. **Context commitments must be cheap.** If committing to the contextual state of a training step costs more than the training step itself, the economics fail. The construction likely requires Merkle commitments, hash chains, or similar amortizable structures on top of existing PoL / zkML primitives, not parallel to them.

3. **Cross-worker consistency is required.** Training protocols cannot verify contextual appropriateness by asking a single worker to report its own context. The construction must cross-reference context claims across a worker set or against a canonical global state published on-chain.

4. **Graceful degradation.** The construction must handle the boundary case where a worker's context is slightly stale but not exploitably so. Binary pass/fail schemes will reject too many honest workers; continuous scoring schemes collapse into reputation in disguise. A *contextual tolerance bound*, expressed in protocol-agnostic terms, is likely a required primitive, and likely has something to learn from the tolerance mechanisms that staleness-aware FL [5, 6, 7] uses algorithmically but at the scheduler layer rather than the verification layer.

5. **Compatibility with existing verification primitives.** Proof-of-context is a layer *on top of* PoL / zkML / TEE, not a replacement. The construction must compose cleanly with all three without requiring changes to their internal cryptographic structure.

The author is actively developing, under the framing above, an initial construction of proof-of-context for a specific class of decentralized-ML protocols. Further publications, code, and benchmark results will follow on the basis of this position paper.

---

## 9. Conclusion

The decentralized-ML protocol stack has absorbed the lesson that computations must be verifiably correct. The federated-learning literature has absorbed the lesson that asynchronous staleness must be *scheduled and weighted*. What has not yet happened, to our knowledge, is the unification: treating context-appropriateness as a single verification-primitive gap that spans training, inference, and prompt-cache, and that has a clear structural precedent in the DeFi oracle-freshness problem.

We publish this paper to (a) establish the term *proof-of-context*, the unified framing, and the cross-domain analogy on the academic record as of 21 April 2026, (b) situate the contribution honestly against the substantial existing work in staleness-aware federated learning and proof-of-learning robustness, and (c) invite protocol teams, independent researchers, and cryptographic-primitive designers to begin treating proof-of-context as a first-class gap in the stack rather than a collection of phase-specific concerns.

The analogy is clean: 2026 decentralized-ML is 2020 DeFi. The next three years of work on the context-verification layer will determine whether the field pays for its stale-oracle equivalents the same way DeFi did.

---

## Acknowledgements

This paper emerged from a 46-day human-AI research collaboration documented in the public narrative *Opus* ([asastuai.github.io/opus](https://asastuai.github.io/opus/)), conducted with Claude (Anthropic) as co-author. The framing crystallized during preparation materials for a 2026 research-engineering application; the term *proof-of-context* was surfaced in collaborative dialogue and subsequently formalized for this paper. The v0.2 revision tightened claims and added the Related Work section after literature review surfaced the substantial body of existing staleness-aware federated learning work; the author is grateful for the intellectual partnership that produced both the naming and the correction.

---

## References

[1] Jia, H., Yaghini, M., Choquette-Choo, C. A., Dullerud, N., Thudi, A., Chandrasekaran, V., & Papernot, N. (2021). *Proof-of-Learning: Definitions and Practice.* IEEE Symposium on Security and Privacy.

[2] Zhang, R., Liu, J., Ding, Y., Wang, Z., Wu, Q., & Ren, K. (2022). *"Adversarial Examples" for Proof-of-Learning.* IEEE Symposium on Security and Privacy. (Demonstrates spoofability of Jia et al. 2021.)

[3] Srivastava, M., Arora, S., & Boneh, D. (2024). *Optimistic Verifiable Training by Controlling Hardware Nondeterminism.* arXiv:2403.09603.

[4] Xing, Z., Wang, J., Wang, Z., et al. (2025). *A Survey of Zero-Knowledge Proof Based Verifiable Machine Learning.* arXiv:2502.18535.

[5] Zhang, W., Gupta, S., Lian, X., & Liu, J. (2016). *Staleness-aware Async-SGD for Distributed Deep Learning.* IJCAI.

[6] Chen, Y., Ning, Y., Slawski, M., & Rangwala, H. (2021). *FedSA: A staleness-aware asynchronous Federated Learning algorithm with non-IID data.* Future Generation Computer Systems.

[7] *Dynamic Staleness Control for Asynchronous Federated Learning in Decentralized Topology.* (2024). Springer LNCS.

[8] *FedPSA: Modeling Behavioral Staleness in Asynchronous Federated Learning.* arXiv:2602.15337.

[9] Wilhelmi, F., Guerra, E., & Dini, P. (2023). *The Implications of Decentralization in Blockchained Federated Learning: Evaluating the Impact of Model Staleness and Inconsistencies.* arXiv:2310.07471.

[10] Fang, L., & Gong, N. Z. (2021). *On the Robustness of Proof-of-Learning.* (Follow-up analysis of PoL spoofability.)

[11] Zheng, Z., Cao, S., & Su, Z. (2023). *Zero-Knowledge Proof-based Verifiable Decentralized Machine Learning in Communication Network: A Comprehensive Survey.* arXiv:2310.14848.

[12] Adams, H., Zinsmeister, N., Salem, M., Keefer, R., & Robinson, D. (2021). *Uniswap v3 Core.* (Time-weighted average price oracle construction and related literature on context verification in DeFi.)

[13] Author's prior work: [github.com/asastuai](https://github.com/asastuai) — Hermetic Computing, intent-cipher, SUR Protocol, LiquidClaw Finance, PayClaw, BaseOracle, TrustLayer.

---

## License and Citation

This paper is released under CC BY 4.0. To cite:

> Maisu, J. C. (2026). *Proof-of-Context: The Missing Verification Layer in Decentralized ML Protocols* (Version 0.2) [Position paper]. github.com/asastuai/proof-of-context, 21 April 2026.

---

## Revision note

**v0.2 (21 April 2026):** Added Section 3 "Related Work" surveying staleness-aware federated learning [5, 6, 7, 8, 9], proof-of-learning robustness literature [1, 2, 3, 10], verifiable-ML surveys [4, 11], and the DeFi oracle-freshness tradition [12]. Tightened abstract and introduction to acknowledge that the underlying problems (staleness, spoofability, nondeterminism) are well-studied in domain-specific contexts and to position the contribution precisely as (a) naming, (b) unification, (c) cross-domain analogy, and (d) extension to attention-based inference. No changes to the four modes of context failure, the DeFi analogy, the prior-art section, or the research program. v0.1 commit remains in git history for priority purposes.
