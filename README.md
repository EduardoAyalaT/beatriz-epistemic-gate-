# Beatriz Epistemic Gate

**An auditable admission-control layer for LLM fine-tuning: verified data trains the model, contradictions are corrected, and unknowns are quarantined.**

**Author:** Eduardo Ayala Tovar • 2026  
**Affiliation:** Independent research / AI sovereignty  
**License:** [PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/)  
**Repository:** https://github.com/EduardoAyalaT/beatriz-epistemic-gate-  
**Documents:** [Whitepaper (EN)](WHITEPAPER_EN.md) • [Whitepaper (ES)](WHITEPAPER_ES.md) • [Corrective Manual](docs/MANUAL_CORRECTIVO.md) • [Funding](FUNDING.md) • [Manifest & hashes](MANIFEST.json)

> **Research status.** Beatriz is a reproducible research prototype. All results below come from controlled, closed-world benchmarks with synthetic attacks authored by the same person who designed the defense. They are **not** a demonstration of general security against real-world data poisoning. Every empirical claim in this README is tied to a specific experiment (EXP01–EXP16) with a SHA-256 hash. Illustrative figures in the Corrective Manual are marked as hypothetical and are not claimed as results.

> **Infrastructure note.** Orchestration ran on a 2006 Toshiba Satellite U205 (2 GB RAM). All model execution used free Kaggle T4 sessions. Direct compute cost: USD $0. This is stated as context for what was and was not feasible, not as a claim about what scaling would cost.

---

## TL;DR

- **Problem.** During fine-tuning, a small fraction of poisoned examples can drive a model to *indifference* between a true claim and its contradiction (`truth_margin ≈ 0`) on the targeted facts, while training loss falls and perplexity looks fine. Aggregate monitors do not see it.
- **Mechanism.** Beatriz sits between the data source and the learner. Every incoming example is checked against a **human-verified, versioned, hashed anchor corpus** and receives one of four verdicts: `VERIFIED`, `CONTRADICTED`, `UNKNOWN`, `INVALID`. Only verified content trains the learner as-is; contradictions are replaced by the anchor and optionally used in a contrastive correction term; unknown or malformed content is **quarantined** for human review instead of entering the training stream.
- **Evidence.** 16 reproducible experiments, 5 architectures (124M → 3.8B), 3 seeds each. In the largest held-out test (EXP16, Phi-3-mini, 30 unseen facts), final truth margin was **+4.19 ± 0.08** with Beatriz vs **+2.45 ± 0.06** without any gate; gate precision/recall 0.93 / 0.80.
- **Key ablation.** The admission gate *alone* — no change to the learner's loss — delivers ~65–74% of the protective effect (EXP15). The contrastive term adds the rest at a higher perplexity cost.
- **Honest limits.** Corpus of 8–36 facts; synthetic attacks; keyword + cached-embedding routing; a measurable utility cost (perplexity rises); one architecture (Pythia) where paraphrase robustness did *not* improve. Details in [Limitations](#limitations-and-negative-findings).

---

## 1. Problem: surgical poisoning is invisible to aggregate metrics

Define, for a verified claim *t* and its contradiction *ℓ*:
truth_margin = mean log P(t) − mean log P(ℓ)
text

A large positive margin means the model strongly prefers the verified statement. A margin near zero means it cannot tell them apart.

Across EXP05–07 and EXP10–16, the unprotected branch (`NONE`) consistently ends training with `truth_margin ≈ 0` — and in several runs slightly negative — on the attacked facts, **even though its training loss decreases monotonically and its neutral-text perplexity often improves or stays flat** (e.g. EXP13: PPL 9.7 → 11.7 while the margin goes from +1.34 → −0.03). The damage is localized to the targeted facts and does not show up in the numbers people normally watch.

---

## 2. Architecture
Incoming data: synthetic, human, external
│
▼
┌───────────────────────┐
│ BEATRIZ EPISTEMIC │ anchor corpus: human-verified,
│ GATE │◄──versioned, SHA-256 + OpenTimestamps
└───────────┬───────────┘
┌─────────────┼──────────────────┐
▼ ▼ ▼
VERIFIED CONTRADICTED UNKNOWN / INVALID
│ │ │
▼ ▼ ▼
train on train on anchor QUARANTINE
example + optional (excluded from training;
contrastive term human review → may be
│ added to anchor corpus)
▼
LEARNER MODEL (LoRA fine-tuning)
│
▼
ledger: adapter_sha256, epoch records, rollback rules R1–R3
text

### 2.1 Components

| Component | Current prototype implementation |
|---|---|
| **Anchor corpus** | 8 facts (EXP07–15); 36 items in EXP16 (6 attacked + 30 held-out). Each item is a `(truth, lie, keywords)` triple. Hashed and timestamped. |
| **Gate routing** | Keyword match → topic; then cosine similarity between mean last-layer hidden state of the candidate and of the anchor `truth` / `lie` (frozen oracle = the base model). `sim_lie > sim_truth ⇒ CONTRADICTED`. No topic match ⇒ `UNKNOWN`. Empty/short ⇒ `INVALID`. Embeddings are pre-cached; decision cost ≈ 0.1 ms. |
| **Admission policy** | `NONE`: everything passes. `GATE_ONLY`: verified/contradicted examples are replaced by the anchor truth; unknowns dropped. `BEATRIZ`: `GATE_ONLY` + contrastive term on contradicted items. |
| **Learner update** | LoRA (r=8, α=16), 0.10–0.24% trainable parameters depending on architecture. |
| **Ledger / rollback** | Per-epoch records; `adapter_sha256` of final LoRA weights; rules R1 (neutral drop < −0.30), R2 (truth drop > 2.0), R3 (unknown rise > 2.0) tested in EXP05. |

### 2.2 Loss
L_total = α • L_CE(anchor_truth) + β • L_truth
L_truth = softplus( m + log P(lie) − log P(truth) )
text

with α = 0.5, β = 1.0, m = 0.5 in EXP13–16. `L_truth` is applied **only** to items the gate marked `CONTRADICTED`. In EXP08 the margin is computed against a frozen reference policy π_ref (log-ratio form).

What the system does **not** claim: it does not determine universal truth. It enforces a narrower, auditable rule — *an example may not modify the learner unless it is supported by, or corrected against, a defined and inspectable anchor corpus.*

---

## 3. Experimental program (EXP01–EXP16)

Sixteen experiments, each with a hashed JSON report. EXP01–08 are calibration on GPT-2; EXP09–16 are LoRA runs across five architectures. All seeds: `[11, 22, 33]`.

| EXP | Model | Purpose | Key result | SHA-256 (report) |
|---|---|---|---|---|
| 01 | GPT-2 124M | Baseline control | Control learns the injected lie | `713ffa62…4966ae` |
| 02 | GPT-2 124M | CONTROL vs FILTER vs PLACEBO (n=84 matched) | Effect is about admission policy, not data quantity | `4b3d424f…05947e` |
| 03 | GPT-2 124M | CONTAMINATED / HARD / EPISTEMIC | −15.73 / −0.80 / **+13.20** | `369c5cec…441c85` |
| 04 | GPT-2 124M | NONE / HARD-filter / REWRITE | −0.01 / +4.43 / **+10.88** — correcting beats dropping | `ed466d51…d44e86` |
| 05 | GPT-2 124M | Z3 consistency + rollback rules | NONE triggers R3 rollback 3/3; Beatriz completes 8 epochs | `97359f04…3626c1` |
| 06 | GPT-2 124M | Open-text stream | NONE 0.00 (indifference); Beatriz +0.57 → +4.23 | `b2e0e62a…559ed5` |
| 07 | GPT-2 124M | Dense-vector gate, 8-fact anchor | NONE −0.27; Beatriz +10.27 | `5e3da9dc…8273483` |
| 08 | GPT-2 124M | Reference-constrained (π_ref) | Beatriz +11.04 | `c93ba4b7…3d61e2` |
| 09 | GPT-2 124M | **LoRA** (0.236%) | NONE +0.14; Beatriz **+3.55** | `f4382f16…6251` |
| 10 | Qwen-2.5-0.5B | LoRA (0.109%) | NONE −0.25; Beatriz **+8.97** | `e894eaf4…c57731` |
| 11 | TinyLlama-1.1B | LoRA (0.102%) | NONE −0.17; Beatriz **+7.87** | `4c0de934…8cbab` |
| 12 | TinyLlama-1.1B | Held-out (n=2) + paraphrase | Held-out inconclusive; paraphrase +2.82 vs +2.04 | `09a1ad45…a43102` |
| 13 | Phi-3-mini 3.8B | LoRA fused-QKV + held-out | Held-out **+5.91** vs +3.57 | `2c48c702…843152` |
| 14 | Pythia-1.4B | LoRA fused-QKV + held-out | Train +8.03; **paraphrase did not improve** | `2d71e2a7…c35f58` |
| 15 | Phi-3-mini 3.8B | **Ablation** NONE / GATE_ONLY / BEATRIZ | Gate alone = ~65–74% of effect | `d95ac8c8…b008cd` |
| 16 | Phi-3-mini 3.8B | **Held-out n=30** + gate confusion matrix | Held-out **+4.19 ± 0.08**; P 0.93 / R 0.80 | `27eda691…4799f` |

Full hashes are in [MANIFEST.json](MANIFEST.json) and the [Appendix](#appendix-integrity-records).

---

## 4. Results in detail (EXP09–EXP16)

All values are mean ± std over 3 seeds at the final epoch (8/8). "Train" = attacked facts; "Held-out" = facts never used as training targets. In EXP09–11 the metric is reported as *semantic margin* in the notebooks; it is computed identically to `truth_margin`.

### 4.1 Cross-architecture summary

| EXP | Architecture | Params | LoRA % | NONE (train) | BEATRIZ (train) | Δ |
|---|---|---:|---:|---:|---:|---:|
| 09 | GPT-2 | 124M | 0.236 | +0.14 ± 0.05 | **+3.55 ± 0.13** | +3.4 |
| 10 | Qwen 2.5 | 0.5B | 0.109 | −0.25 ± 0.11 | **+8.97 ± 0.09** | +9.2 |
| 11 | TinyLlama | 1.1B | 0.102 | −0.17 ± 0.05 | **+7.87 ± 0.16** | +8.0 |
| 14 | Pythia (NeoX) | 1.4B | 0.167 | −0.09 ± 0.07 | **+8.03 ± 0.54** | +8.1 |
| 13/15/16 | Phi-3-mini | 3.8B | 0.123 | −0.03 ± 0.02 | **+10.13 ± 0.07** | +10.2 |

In every architecture, unprotected LoRA fine-tuning ends at or below zero margin on the attacked facts. Beatriz keeps it strongly positive, with seed variance ≤ 0.16 everywhere except Pythia (0.54).

### 4.2 Generalization: held-out and paraphrase

| EXP | Architecture | Held-out n | NONE held-out | BEATRIZ held-out | NONE para | BEATRIZ para |
|---|---|---:|---:|---:|---:|---:|
| 12 | TinyLlama 1.1B | 2 | +1.05 ± 0.24 | +1.33 ± 0.26 | +2.04 ± 0.29 | +2.82 ± 0.34 |
| 13 | Phi-3 3.8B | 2 | +3.57 ± 0.17 | **+5.91 ± 0.07** | +3.41 ± 0.24 | +5.63 ± 0.50 |
| 14 | Pythia 1.4B | 2 | +1.22 ± 0.41 | +2.09 ± 0.11 | **+1.68 ± 0.34** | +1.30 ± 0.07 |
| 16 | Phi-3 3.8B | **30** | +2.45 ± 0.06 | **+4.19 ± 0.08** | +3.41 ± 0.24 | +5.63 ± 0.50 |

Reading: with n=2 the held-out signal is weak on TinyLlama, clear on Phi-3, moderate on Pythia. EXP16 scales the held-out set to 30 facts on Phi-3 and the separation persists (Δ ≈ +1.7, > 20× the pooled std). **EXP16 is the number we consider the conservative headline.** The Pythia paraphrase result is a negative finding (see §6).

### 4.3 Ablation (EXP15, Phi-3-mini, 40-sentence neutral set)

| Branch | Train | Held-out (n=2) | Paraphrase | PPL | Gate ms/call | Peak VRAM |
|---|---:|---:|---:|---:|---:|---:|
| BASE (before FT) | +1.34 | +1.90 | +2.21 | 12.7 | — | — |
| NONE | −0.03 ± 0.02 | +3.57 ± 0.17 | +3.41 ± 0.24 | 30.9 | 0.009 | 7.86 GB |
| GATE_ONLY | +7.46 ± 0.24 | +5.08 ± 0.09 | +5.10 ± 0.51 | 58.8 | 0.102 | 7.86 GB |
| BEATRIZ | +10.13 ± 0.07 | +5.91 ± 0.07 | +5.63 ± 0.50 | 86.3 | 0.107 | 7.97 GB |

- Isolated contribution of the contrastive term (BEATRIZ − GATE_ONLY): **+2.67 train, +0.82 held-out**.
- Share of the total gain (vs NONE) delivered by the gate alone: **~74% train, ~65% held-out** — without modifying the learner's loss function.
- Perplexity cost is additive: fine-tuning on a tiny corpus alone (NONE) already raises PPL 12.7 → 30.9; the gate roughly doubles it; the contrastive term adds another ~28. `GATE_ONLY` has the best protection-per-perplexity ratio in the series.
- 9 runs completed in 15.6 min on a single Tesla T4.

### 4.4 Scaled held-out and gate quality (EXP16, Phi-3-mini)

| Branch | Train | Held-out (n=30) | PPL | Gate precision | Gate recall |
|---|---:|---:|---:|---:|---:|
| NONE | −0.03 ± 0.02 | +2.45 ± 0.06 | 30.9 | — | — |
| GATE_ONLY | +7.46 ± 0.24 | +3.53 ± 0.13 | 58.8 | 0.93 | 0.80 |
| BEATRIZ | +10.13 ± 0.07 | +4.19 ± 0.08 | 86.3 | 0.93 | 0.80 |

Gate confusion (summed over 3 seeds, 1,440 stream draws): TP 535 • FN 133 • FP 39 • TN 733. Base held-out margin before fine-tuning was +1.57, so Beatriz's net held-out gain over base is ≈ +2.6.

### 4.5 Determinism

Train-margin and PPL trajectories for Phi-3 are bit-identical across EXP13, EXP15 and EXP16 under the same seeds (e.g. seed 11, epoch 1: loss 0.8835, train −0.15 in all three). Held-out columns differ only because the held-out set changed (2 → 30). Final LoRA adapters are hashed (`adapter_sha256`) in each report.

---

## 5. Reproducibility

Each experiment folder contains the notebook (`.ipynb`), an exported script (`.py`), the JSON report, and an OpenTimestamps proof (`.ots`).
exp_calibracion_01-07/ EXP01–07 (GPT-2, calibration)
exp08/ reference-constrained (π_ref)
exp09/ LoRA retention, GPT-2
beatriz-epistemic-gate-exp-10-15/ Qwen, TinyLlama, Phi-3, Pythia, ablation
exp16/ scaled held-out + confusion matrix
beatriz-epistemic-gate/ gate implementation
docs/MANUAL_CORRECTIVO.md corrective-training manual (illustrative figures flagged)
text

Verify a bundle:

```bash
sha256sum exp16.rar          # compare with MANIFEST.json
ots verify exp16.rar.ots     # OpenTimestamps proof
Run on Kaggle: open the notebook, enable a T4 GPU, run all cells. Base models download from the Hugging Face Hub (GPT-2 is also included offline, hash c7d00560…20373). Total wall-clock for EXP16 was ~22 min.
________________________________________
6. Limitations and negative findings
We list these ourselves rather than wait for a reviewer to.
1.	Tiny corpus. 8 anchor facts in most experiments, 36 in EXP16. Nothing here shows behaviour at hundreds or thousands of claims.
2.	Closed-world, self-authored attacks. Truth/lie pairs and the poisoning schedule were written by the author. Independently designed attacks, indirect contradictions, partial truths and multilingual inputs are untested.
3.	Prototype routing. Keyword matching + cached hidden-state cosine similarity. The 0.1 ms figure is the decision cost on pre-computed embeddings; it excludes live embedding, retrieval, quarantine handling and any end-to-end production latency.
4.	Utility cost. Perplexity on neutral text rises with each component (EXP15). In EXP12 two of three seeds finished below baseline PPL, so the cost is configuration-dependent, not intrinsic — but it is real and unresolved.
5.	Held-out evidence is architecture-dependent. Clear on Phi-3 (EXP13/16), weak on TinyLlama (EXP12), moderate on Pythia (EXP14).
6.	Negative result on Pythia paraphrases (EXP14). Beatriz scored below the unprotected branch (+1.30 vs +1.68, n=2). Hypothesis to test: on this architecture the contrastive term may bind to anchor wording rather than meaning.
7.	Highest instability on Pythia. Train-margin seed variance 0.54; one seed lost ~1 point in the last epoch.
8.	No external replication yet. All runs are by one person on one type of GPU.
9.	License. PolyForm Noncommercial restricts commercial reuse. Research and educational use is permitted.
________________________________________
7. Roadmap (what funding would change)
Workstream	Now	Next
Anchor corpus	36 hand-written items	Hundreds–thousands of claims with source provenance, versioning, review records
Quarantine	Verdict only (UNKNOWN dropped)	Operational human-in-the-loop review queue; reviewed items promoted into the anchor with audit trail
Routing	Keywords + cached oracle embeddings	Dedicated retrieval encoder (e.g. E5-small), contradiction detection, calibrated uncertainty thresholds, live latency measured honestly
Attacks	Self-authored, synthetic	Independently authored; real-world datasets; paraphrase, partial-truth, multilingual, adversarial
Utility	PPL reported, not optimized	Explicit safety–utility frontier; GATE_ONLY vs BEATRIZ under constraints; downstream task evals
Replication	Single author, Kaggle T4	External replication; standardized benchmark spec and harness released
Governance stack	Ledger fields + rollback rules R1–R3	End-to-end demo: gate → quarantine → review → anchor update → model update → rollback
See FUNDING.md.
________________________________________
8. Sovereign governance context
Beatriz is the training-time enforcement component of a broader architecture whose goal is that model updates be traceable, reviewable and reversible: what data was admitted, on what evidence, which parameters changed, and when a rollback should fire. The broader stack is described in the whitepaper; only the gate and the ledger fields above are implemented and tested in this repository.
________________________________________
Appendix: integrity records
Seeds: [11, 22, 33] • GPT-2 offline model: c7d00560d8910fbed77ffad4065dee5011c41ba401b1064e749c498ba9e20373
EXP	SHA-256
01	713ffa6227b68a9837a11f245b8c4e52d11b343d7f2d0c8f4af916edee4966ae
02	4b3d424f308943ce41c3d6c8f11b8c99130e1eb39fb380ab4f42e7ed1705947e
03	369c5cec029b792744978a9c81434ff46528be5fee49377c621a5f67c9441c85
04	ed466d516acdd202bd4c71d76e6919422d6dff00df019bdd6ac55e618d44ee86
05	97359f0464d7a7f7c33a87bca8a3d9e66b212495f5152c4603105ee1493626c1 (prereg 57691ac0…e69f9e, sanity 2ee4949d…ac116b)
06	b2e0e62a84b1ed19f97657c36308076a44cae42c806b2022bcfd19a55a559ed5
07	5e3da9dca9162f62c6aac94175133e9cbf70ac101389f6714afc24d1b8273483
08	c93ba4b74ecb8ade32f0f645761c1441382c612105bf6b454d5fd9e62a3d61e2
09	f4382f16e5cd1a1877fbcefcd37d98d763b10e054020cb4233295940bd2b6251
10	e894eaf462ca00e40caea0ca13a8eb8df5445bf938d4e2b235bf844110c57731
11	4c0de9343412777ab592073aba999977be86cd23e0bd1bf4fac4cf89a9a8bcab
12	09a1ad451546493c6143788959acff580f47145b5f43a3f87d87e117c7a43102
13	2c48c7020a420ee6adc447efbf0bb668731f48e2bf2f1b51eecab921a9843152
14	2d71e2a7bf24e6132f2f2e7304ec1a91e38e113763bee3029c17a9ef34c35f58
15	d95ac8c87fe13068bb0ffa7c0699a27c8a9d7c915d42094ff57b3cb4b2b008cd
16	27eda691cad2b93c1181556eb2313f0d0f9b0d86e0cb03610989ccc507b4799f
Bundles
File	SHA-256
exp_calibracion_01-07.rar	7c0ba312ec1883b8aab3d54b0493fc0c9a7185087135fb80a6fc966d8b19543b
beatriz-epistemic-gate.rar	54fd6538619d516762ad8a9ab3028b9db0b651ae42216b6d131da358fcaf947a
beatriz-epistemic-gate-exp-10-15.rar	54e2338bce9a15ff8c2a1ef57500dfdcbc4149344749cf4ea3c1301748961018
exp16.rar	9958a3889dffa3dd11d220322da188fdc8347355ef219cd2a43b04932e43a327
________________________________________
Authorship, contact, license
Beatriz Epistemic Gate and the corrective-training protocols were created by Eduardo Ayala Tovar (2026). Copyright notices, license terms and cryptographic hashes are included to establish a public record of authorship.
•	Technical discussion & collaboration: danterunar@yahoo.com
•	Code issues: GitHub Issues on this repository
•	Public evaluation thread: https://arena.ai/c/01a07f4c-3455-756f-ae3e-852f1b0e4804
Licensed under PolyForm Noncommercial License 1.0.0. Non-commercial, research and educational use permitted; commercial use requires the author's written authorization.

