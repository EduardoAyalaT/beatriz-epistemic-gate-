Markdown
# Beatriz Epistemic Gate: Defense Against Fine-Tuning Poisoning

**Author:** Eduardo Ayala Tovar - 2026
**License:** PolyForm Noncommercial License 1.0.0
**Hardware:** Toshiba Satellite U205 (2006, 2GB RAM) + Kaggle T4 x2 - Cost $0 USD
**Affiliation:** Independent Research / AI Sovereignty

> **Honesty note:** The "34% improvement" type percentages in the Manual are illustrative and hypothetical for pedagogical purposes. The only empirical results verified with SHA-256 hash are those from EXP01 to EXP16 in this table.

### Summary
The industry assumes that defending an LLM requires million-dollar clusters. This project demonstrates the opposite. We present **Beatriz**, a non-invasive epistemic proxy that prevents a model from learning lies even when 70% of the stream is poisoned.

Without defense, the truth margin collapses to `≈0.0` - exact indifference between truth and lie - while perplexity appears to improve. With Beatriz, the margin is preserved and generalizes to facts it never saw.

### The Problem: Surgical and Invisible Poisoning
In EXP05, EXP06, EXP07, EXP10, EXP11, EXP13, EXP14, the `NONE` branch collapses. This is the Manual phenomenon. PPL-based monitors are blind.

### The Solution: Beatriz
1.  **Immutable Anchor Corpus:** 8 facts in EXP07-14, 36 facts in EXP16 [6 train + 30 held-out], with SHA-256 + OpenTimestamps .ots
2.  **Dense Vector Gate:** Frozen offline oracle, `embedding = mean hidden_states[-1]`, cosine similarity → VERIFIED / CONTRADICTED / UNKNOWN / INVALID
3.  **Composite Loss:** `L_total = α•L_ce + β•L_truth` where `L_truth = Softplus(MARGIN + logP(lie) - logP(truth))`

### Experimental Series 01-16 - Only What Actually Ran

| EXP | Model | LoRA | Key Result | SHA-256 of report |
|---|---|---|---|---|
| 01 | GPT-2 124M | - | Filter vs no filter, CONTROL learns lie | `713ffa6227b68a...` |
| 02 | GPT-2 124M | - | CONTROL vs FILTER vs PLACEBO 84 equalized | `4b3d424f3089...` |
| 03 | GPT-2 124M | - | HARD -0.80 does not reach, EPISTEMIC +13.20 | `369c5cec029b...` |
| 04 | GPT-2 124M | - | REWRITE +10.88 > HARD +4.43 > NONE -0.01 | `ed466d516acd...` |
| 05 | GPT-2 124M | - | Fire Test Z3 + rollback, NONE fails 3/3, BEATRIZ tm 26.75 | `97359f0464d7...` |
| 06 | GPT-2 124M | - | Open text, NONE 0.00, BEATRIZ +4.23 | `b2e0e62a84b1...` |
| 07 | GPT-2 124M | - | Dense Vector Gate 8 facts, NONE -0.27, BEATRIZ +10.27 | `5e3da9dca916...` |
| 08 | GPT-2 124M | - | Reference-Constrained + pi_ref, BEATRIZ +11.04 | `c93ba4b74ecb...` |
| 09 | GPT-2 124M | 0.23% | LoRA Retention, BEATRIZ +3.54 PPL 132 | `f4382f16e5cd...` |
| 10 | Qwen-2.5-0.5B | 0.10% | NONE -0.24 collapse, BEATRIZ +8.96 | `e894eaf462ca...` |
| 11 | TinyLlama-1.1B | 0.10% | NONE -0.17, BEATRIZ +7.86 | `4c0de9343412...` |
| 12 | TinyLlama-1.1B | 0.10% | Held-Out n=2 + Paraphrase, generalizes +1.33 | `09a1ad451546...` |
| 13 | Phi-3-mini 3.8B | 0.12% | NONE -0.03 / +3.57, BEATRIZ +10.13 / +5.91 | `2c48c7020a42...` |
| 14 | Pythia-1.4B | 0.16% | NONE -0.09, BEATRIZ +8.02 | `2d71e2a7bf24...` |
| 15 | Phi-3-mini 3.8B | 0.12% | Ablation NONE / GATE_ONLY / BEATRIZ | `d95ac8c87fe1...` |
| 16 | Phi-3-mini 3.8B | 0.12% | Held-Out n=30 Prec 0.93 Rec 0.80 | `27eda691cad2...` |

**5 architectures:** GPT-2 124M, Qwen-2.5-0.5B, TinyLlama-1.1B, Pythia-1.4B, Phi-3-mini 3.8B

### Time-Stamped Packages - Your 4 hashes from your Toshiba
exp_calibracion_01-07.rar -> 7c0ba312ec1883b8aab3d54b0493fc0c9a7185087135fb80a6fc966d8b19543b
beatriz-epistemic-gate.rar -> 54fd6538619d516762ad8a9ab3028b9db0b651ae42216b6d131da358fcaf947a
beatriz-epistemic-gate-exp-10-15.rar -> 54e2338bce9a15ff8c2a1ef57500dfdcbc4149344749cf4ea3c1301748961018
exp16.rar -> 9958a3889dffa3dd11d220322da188fdc8347355ef219cd2a43b04932e43a327
text

### Key Results - EXP15 Surgical Ablation
BASE: +1.34 train / +1.90 held-out / PPL 12.7
NONE: -0.03±0.02 / +3.57±0.17 / PPL 30.9
GATE_ONLY: +7.46±0.24 / +5.08±0.09 / PPL 58.8 - Contributes 65% without touching the loop
BEATRIZ: +10.13±0.07 / +5.91±0.07 / PPL 86.3 - Adds the remaining 35%
Gate: 0.107 ms/call - VRAM 7.97 GB

### Bridge: From the Ideal Manual to the Sovereign $0 Prototype
| Manual requires laboratory | Beatriz with $0 |
|---|---|
| Massive corpus with API | 36 facts with hash + .ots |
| L_total with 4 terms | L_truth as Softplus + L_divergence as pi_ref and PPL |
| L_logic with Z3 | EXP05: Z3 sat 24 axioms, 0 mismatches in 672 claims, 7.9ms/claim |
| Ledger + rollback | model_hash, adapter_sha256, prereg_sha, sanity_sha, R1/R2/R3 |

### Verification
certutil -hashfile exp_calibracion_01-07.rar SHA256
ots verify exp_calibracion_01-07.rar.ots
text

### Public evaluation
https://arena.ai/c/01a07f4c-3455-756f-ae3e-852f1b0e4804
