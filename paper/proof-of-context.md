# Proof-of-Context: The Missing Verification Layer in Decentralized ML Protocols

**Juan Cruz Maisu**
Buenos Aires, Argentina · juancmaisu@outlook.com · github.com/asastuai

**Version 0.1** — **Published: 21 April 2026**

---

## Abstract

Decentralized machine-learning protocols have made significant progress on two verification primitives: *proof-of-learning* in distributed training marketplaces (e.g., Gensyn) and *zkML proofs paired with TEE attestations* in verifiable-inference networks (e.g., OpenGradient). Both verify that a computation was executed correctly. Neither verifies that the worker was operating in a contextually appropriate state. This position paper names that gap — **proof-of-context** — and argues that its absence is the structural analogue, in ML protocols, of the oracle-freshness problem that accounts for the majority of value-stolen exploits in decentralized finance. We identify four recurring modes of context failure, draw the DeFi analogy explicitly, and sketch a research program for building proof-of-context primitives without yet specifying a cryptographic construction. This document establishes the term and framing as of its publication date for purposes of academic priority.

**Keywords:** verifiable computing · decentralized ML · proof-of-learning · zkML · oracle problem · adversarial protocols · cryptoeconomic design

---

## 1. Introduction

The emerging class of decentralized ML protocols — training marketplaces, verifiable-inference networks, agent-compute exchanges — have inherited a specific verification posture from their cryptographic roots. The posture is: *prove that the computation happened, prove that it produced the claimed output.* Proof-of-learning does this for training steps. zkML proofs do this for inference. TEE attestations do this for hardware execution. All three are real, load-bearing, and hard-won.

This paper argues that this posture, while necessary, is *not sufficient*. Verifying that a computation was executed correctly is a different and weaker guarantee than verifying that the computation was executed *in the right context*. We name the latter gap **proof-of-context** and argue that closing it is the next structural problem in the decentralized-ML stack.

The motivating observation is cross-domain. In decentralized finance, the analogous gap — the oracle-freshness problem — has been responsible for the majority of value-stolen exploits across every major AMM, lending protocol, and derivatives platform. Systems that verified "did the swap execute correctly?" failed because they did not verify "was the price you swapped against fresh?" We expect the same pattern to appear in decentralized ML, and we believe it is already latent in production systems.

---

## 2. Background: Existing Verification Primitives

We restrict attention to three primitives currently deployed in decentralized ML protocols.

**Proof-of-learning.** In protocols such as Gensyn, a worker commits on-chain to intermediate gradient checkpoints at agreed intervals. Verifiers re-run short segments of the training step against the submitted checkpoints and confirm that the gradients match. The guarantee: *the reported gradient was produced by the claimed training operation on the claimed input.*

**zkML proofs.** In protocols such as OpenGradient, a worker produces a zero-knowledge proof that a specific model, identified by its weight commitment, was executed on a specific input and produced a specific output. The guarantee: *the reported inference is cryptographically bound to the claimed model + input pair.*

**TEE attestations.** In the same class of protocols, a Trusted Execution Environment produces a signed attestation that the computation occurred inside the enclave, that the enclave ran the claimed code, and that the output was produced by that code. The guarantee: *the hardware executed the claimed binary honestly.*

Each of these primitives verifies a dimension of computational correctness. Together they cover much of the attack surface a malicious worker might exploit to fabricate work. What they do not cover is the question of whether the correct computation, executed on the correct input, was nonetheless performed under the *wrong contextual state.*

---

## 3. The Gap: Computation Correctness vs. Context Appropriateness

Machine-learning training and inference, unlike a token swap, are not memoryless. Every gradient step in a training run depends on the global state of the model at that exact moment — the current weights, the current optimizer state, the current learning-rate schedule, the correct data shard at the correct step. Every inference call depends on the current model version, the current prompt cache (for attention-based systems), the current system prompt, the current set of tool bindings, and so on. The computation is only meaningful in the context in which it was performed.

A worker can produce a mathematically correct gradient on a stale model snapshot. The gradient passes proof-of-learning verification trivially — the math is right. But the gradient is globally useless, or worse: it actively degrades the training run because it pulls the optimizer backward relative to other workers operating on the fresh snapshot. The primitive passes, the protocol pays, the model gets worse.

A worker can produce a valid zkML proof of an inference call on a stale model version. The proof passes verification trivially — the computation on the committed weights is correct. But the client received an answer from a deprecated model. The primitive passes, the protocol pays, the user is silently served degraded intelligence.

The generalization: **a computation can be correct and still be wrong**, if the context under which it was performed is not the context the protocol intended.

---

## 4. The DeFi Analogy

Decentralized finance has lived with exactly this gap for six years.

Early AMMs verified that swaps executed according to their bonding curves. The swap itself was always correct: given a pool state, the curve produced the deterministically correct output amount. What AMMs did not verify — and what flash-loan attackers exploited repeatedly — was whether the *pool state itself* reflected fresh price information at the moment of the swap. Attackers temporarily manipulated the pool state (via flash loans), swapped against the manipulated state (with computational correctness), and exited. Every swap passed the protocol's verification primitive. Billions of dollars were drained.

Lending protocols tracking collateral value via on-chain oracles suffered the same pattern under the name *stale oracle exploits*. The price feed was occasionally late by one block or one epoch, and attackers used the delay to liquidate healthy positions or borrow against undervalued collateral. The feed was never wrong in the sense of reporting a fabricated number. It was wrong in the sense of reporting a true number *from the wrong moment*.

The DeFi response to this gap, over several years, was the construction of a full secondary verification layer around *context verification* — freshness, deviation thresholds, time-weighted averages, multiple-source quorums, circuit breakers. The primary verification of computational correctness stayed where it was. But on top of it, the industry built a context layer that became load-bearing for safety.

**Claim:** The decentralized-ML protocol stack is, in 2026, roughly where DeFi was in 2020 on this axis. The primary verification layer (PoL, zkML, TEE) is hardening quickly. The context verification layer does not yet exist as a named component. The gap is being paid for silently — in degraded training runs, in deprecated-model inferences, in rewards flowing to workers who computed correctly but contributed nothing useful.

---

## 5. Modes of Context Failure

Four recurring modes, all of which pass existing verification primitives while violating the protocol's practical intent:

**C1. Stale model snapshot.** A training worker computes gradients against a model version that is multiple global updates behind. Each individual gradient is correct relative to its local starting point. Aggregated into the global optimizer, the gradient pushes the model toward a previous state. An inference worker serves a deprecated model version. Each output is correct for the weights used. The client receives outdated intelligence.

**C2. Data-shard desynchronization.** A training worker receives a data shard that was intended for a different training step or a different curriculum phase. The gradient is mathematically correct on the data received. It is globally harmful because it represents the wrong curriculum signal at the wrong moment of training.

**C3. Optimizer-state drift.** A worker computes gradients with an optimizer state (momentum terms, second moments, adaptive learning rates) that is out of sync with the global optimizer. The gradient math is correct. The global optimizer update it produces is not — the momentum component is wrong relative to the global trajectory.

**C4. Prompt-cache / context-window staleness.** In inference for attention-based models, a worker operates against a stale prompt cache or a stale system prompt. The inference call is computationally correct against the cached state. The user receives an output conditioned on outdated context.

All four are invisible to PoL, zkML, and TEE in their current forms. All four degrade the protocol's practical output. All four have direct DeFi analogues in stale-oracle and state-lag exploits.

---

## 6. Prior Art from the Author's Portfolio

Two artifacts from the author's production work (2025–2026) anticipated components of this problem from the agent-infrastructure side, prior to formalizing the proof-of-context framing:

**TrustLayer** ([github.com/asastuai/TrustLayer](https://github.com/asastuai/TrustLayer)) — An agent-reputation infrastructure layer on Base L2 combining skill audit, test harness, uptime monitoring, and payment escrow into a single x402-native API. TrustLayer's reputation score is a rough historical sketch of *contextually-appropriate behavior over time* — the dimension that proof-of-context systems would formalize into a real-time verification primitive rather than an after-the-fact scoring mechanism.

**BaseOracle** ([github.com/asastuai/BaseOracle](https://github.com/asastuai/BaseOracle)) — A pay-per-query data oracle for autonomous AI agents on Base, priced via x402 micropayments. BaseOracle enforces contextual freshness on a per-query basis at the data layer and embeds it in the payment primitive; the same structural insight — *freshness must be verified at pay-time, not discovered retroactively* — is the insight proof-of-context generalizes to the compute layer.

Neither artifact is a solution to the proof-of-context problem as formulated here. Both are prior evidence that the author was circling the same structural gap from an adjacent angle before naming it.

---

## 7. A Research Program

We do not in this paper propose a cryptographic construction for proof-of-context. That construction is the central open research question the framing invites. We do, however, sketch the constraints any viable construction must satisfy:

1. **Freshness is verifiable per-computation, not reputational.** The protocol must be able to reject a specific gradient / inference / contribution for contextual staleness without relying on long-horizon reputation scores. (Cf. the DeFi evolution away from "trusted reporter" oracle models and toward per-query freshness proofs.)

2. **Context commitments must be cheap.** If committing to the contextual state of a training step costs more than the training step itself, the economics fail. The construction likely requires Merkle commitments, hash chains, or similar amortizable structures on top of existing PoL / zkML primitives, not parallel to them.

3. **Cross-worker consistency is required.** Training protocols cannot verify contextual appropriateness by asking a single worker to report its own context. The construction must cross-reference context claims across a worker set or against a canonical global state published on-chain.

4. **Graceful degradation.** The construction must handle the boundary case where a worker's context is slightly stale but not exploitably so. Binary pass/fail schemes will reject too many honest workers; continuous scoring schemes become reputation in disguise. A *contextual tolerance bound*, expressed in protocol-agnostic terms, is likely a required primitive.

5. **Compatibility with existing verification primitives.** Proof-of-context is a layer *on top of* PoL / zkML / TEE, not a replacement. The construction must compose cleanly with all three without requiring changes to their internal cryptographic structure.

The author is actively developing, under the framing above, an initial construction of proof-of-context for a specific class of decentralized-ML protocols. Further publications, code, and benchmark results will follow on the basis of this position paper.

---

## 8. Conclusion

The decentralized-ML protocol stack has absorbed the lesson that computations must be verifiably correct. It has not yet absorbed the lesson — taught painfully over six years in DeFi — that *correct computations in wrong contexts* form their own attack surface, and that this attack surface does not close itself. Proof-of-context, as a named verification layer, is the framing this paper proposes for that problem.

We publish this paper now to (a) establish the term, the framing, and the research program on the academic record as of 21 April 2026, and (b) invite protocol teams, independent researchers, and cryptographic-primitive designers to begin treating proof-of-context as a first-class gap in the stack rather than a collection of ad-hoc oversights.

The analogy is clean: 2026 decentralized-ML is 2020 DeFi. The next three years of work on the context-verification layer will determine whether the field pays for its stale-oracle equivalents the same way DeFi did.

---

## Acknowledgements

This paper emerged from a 46-day human-AI research collaboration documented in the public narrative *Opus* ([asastuai.github.io/opus](https://asastuai.github.io/opus/)), conducted with Claude (Anthropic) as co-author. The framing crystallized during preparation materials for a 2026 research-engineering application; the term *proof-of-context* was surfaced in collaborative dialogue and subsequently formalized for this paper. The author is grateful for the intellectual partnership that produced the naming and for the body of prior work (TrustLayer, BaseOracle, Hermetic Computing) that provided the substrate the framing grew out of.

---

## References

Primary references are intentionally left minimal at this position-paper stage. Future iterations will incorporate the full literature on verifiable compute, cryptographic voting protocols, DeFi oracle design, and reputation systems.

- Gensyn Protocol documentation (2024–2026) — [gensyn.ai](https://gensyn.ai)
- OpenGradient protocol overview (2024–2026) — [opengradient.ai](https://www.opengradient.ai/)
- Sahai, A., & Waters, B. (2005). *Fuzzy identity-based encryption.* (Referenced for the broader context of policy-bound cryptographic primitives.)
- Author's prior work: [github.com/asastuai](https://github.com/asastuai) — Hermetic Computing, intent-cipher, SUR Protocol, LiquidClaw Finance, PayClaw, BaseOracle, TrustLayer.

---

## License and Citation

This paper is released under CC BY 4.0. To cite:

> Maisu, J. C. (2026). *Proof-of-Context: The Missing Verification Layer in Decentralized ML Protocols* (Version 0.1) [Position paper]. github.com/asastuai/proof-of-context, 21 April 2026.
