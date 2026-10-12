# Gravit Core — Epistemic Verification Layer

[![Tests](https://github.com/gravit-open-network-foundation/gravit-core/actions/workflows/ci.yml/badge.svg)](https://github.com/gravit-open-network-foundation/gravit-core/actions)
[![License](https://img.shields.io/badge/license-Apache_2.0-green)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)]()

**Research prototype of a structural verification gate for AI-agent actions.**
**The LLM advisory layer cannot override the decision — the test below shows it.**

> The reference implementation of the Gravit specifications (VEDS-00, GEVP) is the
> [Gravit Verifier](https://github.com/gravit-open-network-foundation/verifier); every published
> figure is reproduced by the [evaluation pack](https://github.com/gravit-open-network-foundation/gravit-eval-pack).

## What makes Gravit different?

| Feature | How it works | Why it matters |
|---------|--------------|----------------|
| **No self-attestation** | `VerificationRequest` has no field for agent confidence | Agent cannot "claim" trust — it must be earned through reasoning structure |
| **LLM cannot override** | Structural gate decides; advisory layer is isolated | Even a "rogue" LLM cannot force a bad decision |
| **Deterministic audit proofs** | `trace_id` = SHA-256 over all inputs | Any third party (regulator, court) can recompute and verify |
| **Hard policy rules** | KYC, sanctions, limits enforced before trust score | Regulatory compliance is not optional |

## Quick start (30 seconds)

```bash
# Clone and run
git clone https://github.com/gravit-open-network-foundation/gravit-core
cd gravit-core
docker-compose up --build

# In another terminal, verify a request
curl -X POST http://localhost:8080/v1/verify \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-key-do-not-use-in-production" \
  -d '{
    "reasoning_chain": [
      {"text": "User authorized transfer to 0x123", "source_type": "user_input", "action_type": "TRANSFER"}
    ],
    "context": {"kyc_verified": true},
    "proposed_action": "TRANSFER"
  }'
```

Expected response:

```json
{
  "decision": "ALLOWED",
  "trust_score": 0.92,
  "truth_vector": {"anchor_integrity": 1.0, "dependency_validity": 1.0, "policy_compliance": 1.0, "adversarial_risk": 0.23},
  "trace_id": "a1b2c3...",
  "reason": "trust_score=0.92"
}
```

## What the tests check

> **Correction (12 Oct 2026).** An earlier version of this table showed "p95 latency 47 ms",
> "attack success rate 0.7 %" and "false positive rate 1.2 %". The tests below do not compute
> those figures, so they have been removed; the earlier text remains in the history of this
> repository.

| Test | What it actually checks | How to run |
|------|-------------------------|------------|
| benchmarks/test_latency.py | p95 latency of one simple request over 1000 runs stays below 50 ms (measured on a 2024 Mac: 0.02 ms; depends on hardware) | `pytest -s benchmarks/test_latency.py` |
| benchmarks/test_adversarial.py | one replay attack (duplicated transfer step) is not ALLOWED | `pytest benchmarks/test_adversarial.py` |
| benchmarks/test_false_positive.py | one legitimate request, evaluated 1000 times, is never DENIED (the gate is deterministic, so this checks one case, not a rate) | `pytest -s benchmarks/test_false_positive.py` |
| tests/test_verifier.py | the LLM advisory layer cannot change the decision | `pytest tests/test_verifier.py -k test_llm_advisory_cannot_override_decision` |

These are functional tests of single cases, not statistical benchmarks. For an adversarial
benchmark of the reference verifier see the
[evaluation pack](https://github.com/gravit-open-network-foundation/gravit-eval-pack).

## Proof that LLM cannot override (run this yourself)

```bash
pytest tests/test_verifier.py -k test_llm_advisory_cannot_override_decision -v
```

This test creates an advisory layer that explicitly says "APPROVE" with 99% confidence against a reasoning chain that violates a hard policy rule (no KYC). The result is still DENIED, and the TruthVector is bit-for-bit identical to running without the advisory layer.

If you can make the advisory layer change the decision, we will publicly admit defeat and rewrite our system. You cannot.

## Public testnet

🚀 Public testnet launching Q3 2026

For early access (enterprise, researchers, regulators):
📧 testnet@gravitnetwork.org | 🐦 @GravitNet on X

## Documentation

| Document | Audience | Link |
|----------|----------|------|
| Phase 1 Specification | IETF, regulators, researchers | docs/GRAVIT_Phase1_Specification.md |
| API Reference | Developers, integrators | docs/API_Reference.md |
| MiCAR Compliance Matrix | BaFin, FINMA, ESMA | docs/MiCAR_Compliance_Matrix.md |

> These three documents are placeholders and are still empty. For the specifications see the IETF
> Internet-Drafts [GEVP](https://datatracker.ietf.org/doc/draft-gravit-gevp/) and
> [VEDS](https://datatracker.ietf.org/doc/draft-gravit-verifiable-epistemic-decision/).

## Architecture (simplified)

```
┌─────────────┐     ┌───────────────────────────────────────────────────────────────┐
│ User/AI     │     │                      GRAVIT CORE                              │
│ Agent       │─────│  ┌─────────────────────────────────────────────────────────┐  │
│ (any LLM)   │     │  │                   API (FastAPI)                         │  │
└─────────────┘     │  │  - API key validation (constant-time)                   │  │
                    │  │  - Request/response schemas (no self-attestation)       │  │
                    │  └─────────────────────┬───────────────────────────────────┘  │
                    │                        │                                      │
                    │                        ▼                                      │
                    │  ┌─────────────────────────────────────────────────────────┐  │
                    │  │           STRUCTURAL GATE (Core Decision)               │  │
                    │  │  - Computes TruthVector from reasoning structure        │  │
                    │  │  - Enforces hard policy rules (KYC, sanctions)          │  │
                    │  │  - Detects adversarial patterns (replay, sandwich)      │  │
                    │  │  - NO LLM input, NO agent self-report                   │  │
                    │  └─────────────────────┬───────────────────────────────────┘  │
                    │                        │                                      │
                    │            ┌───────────┴───────────┐                          │
                    │            ▼                       ▼                          │
                    │  ┌─────────────────┐     ┌───────────────────────────────┐    │
                    │  │  Advisory Layer │     │  Audit Proof (SHA-256)        │    │
                    │  │  (LLM, optional)│     │  - Deterministic trace_id     │    │
                    │  │  - Logs only    │     │  - Recomputable by third party│    │
                    │  │  - Cannot affect│     │  - Immutable audit trail      │    │
                    │  │    decision     │     └───────────────────────────────┘    │
                    │  └─────────────────┘                                          │
                    └───────────────────────────────────────────────────────────────┘
```

The Structural Gate is the only component that computes trust_score and decision. The advisory layer is isolated — it can provide context but its output never enters the decision function.

## Comparison with other approaches

> **Removed (12 Oct 2026).** An earlier version compared Gravit Core with products of other
> companies. Those statements had not been verified, so the table was removed; it remains in the
> history of this repository.

## Roadmap

| Phase | Description | Status | Estimated completion |
|-------|-------------|--------|----------------------|
| Phase 1 | Epistemic Verification Layer (structural gate + audit) | ✅ DONE (this repo) | June 2026 |
| Phase 2 | Claim/Anchor Graph (semantic dependencies) | ⚠️ Spec complete, code in progress | Q3 2026 |
| Phase 3 | Multi-agent consensus (GRTVP) | 📝 Design phase | Q4 2026 |
| Phase 4 | ZK-proof integration for private verification | 📝 Research phase | Q1 2027 |
| Phase 5 | Full GON Continuum integration | 📝 Planning | Q2 2027 |

## Contributing

We welcome contributions from researchers, engineers, and regulators. See CONTRIBUTING.md.

## License

Apache 2.0 — free for commercial and research use.

## Contact

- **GitHub:** [gravit-open-network-foundation](https://github.com/gravit-open-network-foundation)
- **X (Twitter):** [@GravitNet](https://twitter.com/GravitNet)
- **LinkedIn:** [Gravit Open Network Foundation](https://linkedin.com)
- **Substack:** [Dr. Alex Konviser](https://substack.com)
- **Email:** info@gravitnetwork.org (general), testnet@gravitnetwork.org (testnet access)
