# Proof-of-Context

### A position paper on the missing verification layer in decentralized ML protocols

**Author:** Juan Cruz Maisu · Buenos Aires, Argentina
**Version 0.2** · **Published: 21 April 2026** *(revised from v0.1 same day — expanded Related Work, tightened claims)*

---

## TL;DR

Decentralized ML protocols verify that *computations are correct*. A substantial literature in federated learning addresses *model staleness* as a scheduling and weighting concern. Neither unifies these into a *verification-primitive layer* that would let the protocol refuse to pay for contextually-inappropriate contributions at contribution time. This paper names the unified gap **proof-of-context**, maps its structural analogue to the DeFi oracle-freshness problem (2020–2024), and sketches constraints for a viable construction. The contribution is the naming, the unification, the cross-domain analogy, and the extension from training into attention-based inference — not the discovery that staleness exists, which is well-studied.

---

## Read

- **Paper (Markdown):** [paper/proof-of-context.md](paper/proof-of-context.md)
- **Paper (PDF):** [paper/proof-of-context.pdf](paper/proof-of-context.pdf)

---

## Citation

```bibtex
@misc{maisu2026proofofcontext,
  title={Proof-of-Context: The Missing Verification Layer in Decentralized ML Protocols},
  author={Maisu, Juan Cruz},
  year={2026},
  month={4},
  day={21},
  howpublished={Position paper, \url{https://github.com/asastuai/proof-of-context}},
  note={Version 0.2}
}
```

---

## Status

This is a **position paper** — it names a gap, maps its DeFi analogue, and enumerates constraints that any viable proof-of-context construction must satisfy. It does *not* propose a specific cryptographic construction. Follow-up work will publish implementation drafts, benchmark results, and protocol-specific adaptations.

Issues, corrections, and co-authorship inquiries are welcome via GitHub Issues or direct contact.

---

## Related work (from the author)

- [Hermetic Computing (kybalion)](https://github.com/asastuai/kybalion) — Rust framework formalizing the Seven Hermetic Principles as computational primitives
- [intent-cipher](https://crates.io/crates/intent-cipher) — Research crate exploring intent-keyed stream ciphers (published v0.2 after public v0.1 reframe)
- [TrustLayer](https://github.com/asastuai/TrustLayer) — Agent reputation infrastructure on Base (prior art for the proof-of-context framing)
- [BaseOracle](https://github.com/asastuai/BaseOracle) — Pay-per-query data oracle for AI agents via x402 (prior art for per-query contextual verification)
- [Opus narrative](https://asastuai.github.io/opus/) — 46-day human-AI research collaboration that produced the body of work this framing grew out of

---

## License

[CC BY 4.0](LICENSE) — free to reuse with attribution.

---

## Contact

Juan Cruz Maisu · [juancmaisu@outlook.com](mailto:juancmaisu@outlook.com) · [github.com/asastuai](https://github.com/asastuai) · [linkedin.com/in/juan-cruz-maisu-b4b610308](https://www.linkedin.com/in/juan-cruz-maisu-b4b610308/)
