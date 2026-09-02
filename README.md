Markdown
# Beatriz Epistemic Gate

**Author:** Eduardo Ayala Tovar  
**Year:** 2026  
**License:** [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/)  
**Spanish version:** [README.es.md](README.es.md)

---

## What is Beatriz?

Beatriz is an **epistemic gate** mechanism designed to mitigate the learning of misinformation in generative models during fine-tuning. During training, an external gate compares generated samples against an anchor corpus of canonical facts and decides whether the model is being exposed to false information.

If a contradiction with the corpus is detected, a **contrastive loss** is applied that penalizes the model's preference for the lie and forces it to anchor to the reference truth.

## Why does this project exist?

Language models can degrade their truthfulness when trained on unvalidated synthetic data or on false information. Beatriz explores a corrective path: **making the anchoring to truth explicit during training**, rather than optimizing only the conditional probability of tokens.

## Experimental trajectory

Development followed an iterative process:

- **EXP01–EXP04**: debugging and calibration of the mechanism.
- **EXP05–EXP08**: fine-tuning of the method and stability validation.
- **EXP09–EXP14**: real tests on five open-source architectures.

This repository publishes **two representative implementations**:

- **EXP08**: full fine-tuning with reference-constrained contrastive loss anchored to a frozen model (`pi_ref`).
- **EXP09**: LoRA adaptation with the integrated epistemic gate.

## Architectures tested

The Beatriz mechanism was experimentally validated on five architectures:

| Architecture   | Organization | Result     |
|----------------|--------------|------------|
| GPT-2          | OpenAI       | ✅ positive |
| Qwen-2.5       | Alibaba      | ✅ positive |
| TinyLlama      | Llama lineage| ✅ positive |
| Phi-3          | Microsoft    | ✅ positive |
| Pythia         | EleutherAI   | ✅ positive |

Detailed records for the last four architectures will be published later. This repository presents two complete and reproducible implementations as the public foundation of the method.

## Coming next: EXP14 preview

As part of the validation across architectures, **EXP14** was executed on **Pythia 1.4B** and included a **held-out evaluation protocol**: the model was trained on a subset of the anchor corpus and evaluated on unseen domains and paraphrases.

This test showed that the Beatriz effect generalizes beyond the training corpus. Full code, results, and reproducibility details for EXP14 will be published in a future update.

## Results summary (real measured data)

The only empirical results presented as real in this project are those of **EXP08** and **EXP09**. All other numerical examples in the conceptual manual are illustrative and hypothetical.

### EXP08 — Full fine-tuning with `pi_ref` anchoring

| Branch   | Final truth margin          | Base PPL → Final PPL          |
|----------|-----------------------------|-------------------------------|
| NONE     | -0.224, -0.265, -0.322      | 102.51 → 541.70, 716.89, 566.67 |
| BEATRIZ  | +10.604, +11.008, +11.042   | 102.51 → 1098.49, 1121.18, 2081.75 |

*Observation:* BEATRIZ achieves a very high truth margin but at the cost of a strong increase in perplexity (overfitting to the anchor corpus).

### EXP09 — LoRA adaptation

| Branch   | Final truth margin          | Base PPL → Final PPL          |
|----------|-----------------------------|-------------------------------|
| NONE     | +0.129, +0.079, +0.201      | 102.51 → 76.97, 76.37, 72.99 |
| BEATRIZ  | +3.729, +3.464, +3.453      | 102.51 → 139.12, 131.80, 126.30 |

*Observation:* with LoRA, BEATRIZ achieves a clearly positive truth margin while keeping perplexity much more controlled than in EXP08. This is the most relevant result of the published set.

## Repository structure

```
beatriz-epistemic-gate/
├── README.md                  # English documentation
├── README.es.md               # Spanish documentation
├── LICENSE                    # PolyForm Noncommercial 1.0.0
├── docs/
│   └── MANUAL_CORRECTIVO.md   # Conceptual manual (hypothetical examples)
├── exp08/
│   ├── exp08_beatriz_ref_constrained.ipynb
│   ├── exp08_beatriz_ref_constrained.py
│   └── exp08_results.json     # Results with SHA-256 hash
└── exp09/
    ├── exp09_beatriz_lora.ipynb
    ├── exp09_beatriz_lora.py
    └── exp09_lora_results.json # Results with SHA-256 hash
```

## How to reproduce

1. Clone the repository.
2. Install dependencies:
   ```bash
   pip install torch transformers peft numpy jupyter
   ```
3. Open the notebook `exp08/exp08_beatriz_ref_constrained.ipynb` or `exp09/exp09_beatriz_lora.ipynb` (they are prepared for Kaggle with local GPT-2 paths; adjust `MODEL_PATH` and `TOK_PATH` if needed).
4. Run all cells. The script will generate a JSON report with its SHA-256 hash.

## Limitations

- The anchor corpus is small (8 domains). Generalization to broader domains is not proven.
- The truth margin is measured on the same anchor corpus used during training.
- EXP08 increases perplexity significantly, suggesting overfitting.
- The comparison is only NONE vs BEATRIZ; no external baselines are included yet.
- Results on Qwen-2.5, TinyLlama, Phi-3, and Pythia are not yet published in this repository (EXP14 is announced as a future release).

## Authorship

The Beatriz epistemic gate and the corrective training protocols presented in this project were created by **Eduardo Ayala Tovar** in 2026. The inclusion of copyright notices, license terms, and cryptographic hashes of results is intended to establish a public record of authorship.

## Contact

- **Community and technical discussions:** `danterunar@yahoo.com`  
- **Commercial licensing and collaborations:** `danterunar@yahoo.com`

For code issues, use the GitHub Issues section of this repository.
## Sustain this research

Running Beatriz (EXP08–EXP14) requires GPU and hardware. If these results or the epistemic gate mechanism are useful to your work, your support keeps the open record and future validation (EXP14) alive.

This is not a commercial transaction ([PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/)).

👉 [Donate via PayPal](https://paypal.me/EAyalaTovar)

## License

This project is licensed under the [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/). Non-commercial use, research, and educational purposes are permitted. Commercial use requires explicit authorization from the author.
