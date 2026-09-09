Beatriz Epistemic Gate: Technical Specification and Defense Architecture against Data Poisoning in Fine-Tuning
Author: Eduardo Ayala Tovar
Year: 2026
License: PolyForm Noncommercial License 1.0.0
Affiliation: Independent Research / AI Sovereignty
Orchestration Hardware: Toshiba Satellite U205 (2006, 2 GB RAM)
Experimental Compute Engine: Kaggle T4 x2 (Total cost: $0 USD) 
________________________________________
Abstract
The contemporary artificial intelligence industry sustains the dogma that advanced research in model safety, alignment, and defense requires massive infrastructure and multi-million-dollar budgets. This paper demonstrates the contrary by presenting Beatriz, a lightweight epistemic gate architecture designed to prevent surgical data poisoning during fine-tuning. Through a series of 16 systematic experiments (EXP08–EXP16) validated across 5 architectures (from 124M GPT-2 to 3.8B Phi-3-mini-4k-instruct), we demonstrate that epistemic poisoning is invisible to conventional aggregate metrics, but can be effectively counteracted by introducing a non-invasive defensive proxy between the generative source and the learning model. Results confirm that filtering based on a static anchor corpus provides 65% of the defensive benefit without altering the student training loop, while an additional contrastive loss term (Softplus) consolidates a robust truth margin (+4.19 ± 0.08 on a scaled held-out set of 30 multi-domain facts), maintaining bit-exact determinism and a decision latency of ~0.1 ms per call.
________________________________________
1. Introduction: The Myth of Massive Infrastructure
Current development of Large Language Models (LLMs) is dominated by resource concentration. It is commonly assumed that auditing vulnerabilities, training counter-fires, or studying representational collapse dynamics requires enterprise clusters.
This research stems from a contrary premise: methodological rigor, architectural clarity, and code sovereignty surpass brute capital force. All orchestration, design, and analysis infrastructure for this experimental series was executed on a conventional 2006 laptop (Toshiba Satellite U205, 2 GB RAM), utilizing free public GPU resources (Kaggle T4 x2) with a total monetary cost of zero dollars ($0).
The objective of this document is to expose the technical architecture and definitive results of the experimental series, providing independent developers and small startups with a portable, low-cost defensive mechanism to shield their training pipelines.
________________________________________
2. Theoretical Framework: Surgical and Invisible Poisoning
In early iterations of the series (EXP12–EXP14), a critical phenomenon in LLM security was documented: malicious epistemic poisoning is surgical and invisible by design.
When an attacker injects specific misinformation into the training stream, the model selectively destroys the attacked facts down to a condition of exact indifference (train margin ≈0.0≈0.0), while global aggregate metrics (such as general perplexity or performance on other tasks) improve or remain stable due to generic fine-tuning on fluent prose.
Traditional monitoring systems based on aggregate statistics are blind to this type of attack. Therefore, an intervention at the training architecture level is required: a mechanism capable of independently verifying objective truth regardless of traditional loss function collapse (Cross-Entropy).
________________________________________
3. System Architecture: The Epistemic Gate
To solve this problem without forcing a restructuring of the complex optimization loops of base models, Beatriz was designed as an epistemic gate structured under a two-layer defensive proxy pattern:
1.	The Immutable Anchor Corpus: A versioned set of canonical facts, separated from the model's training flow, acting as an objective source of truth.
2.	The Dense Vector Gate (DenseVectorGateCached): A low-latency component that evaluates the generated candidate text, compares it against the corpus via cosine similarity in embedding spaces (calculated through a frozen oracle), and issues a verdict (VERIFIED, CONTRADICTED, UNKNOWN, INVALID).
3.1. Composite Loss Function
In its advanced branch (Beatriz), the system operates by modifying optimization through a composite loss:
Ltotal=α⋅Lce+β⋅LcontrastivaLtotal=α⋅Lce+β⋅Lcontrastiva
•	LceLce (α=0.5α=0.5): Standard maximum likelihood loss (Cross-Entropy) to preserve linguistic fluency.
•	LcontrastivaLcontrastiva (β=1.0β=1.0): Term based on a Softplus function over the margin between the log probabilities of the truth and the detected falsehood:
Lcontrastiva=Softplus(Margin+log⁡P(False)−log⁡P(True))Lcontrastiva=Softplus(Margin+logP(False)−logP(True))
This term does not saturate as long as the margin is finite, maintaining active pressure on the model even after cross-entropy is exhausted.
________________________________________
4. Experimental Methodology
The experimental series spanned multiple open-source architectures to demonstrate the universality of the phenomenon:
•	Evaluated Models (EXP01–EXP14): GPT-2 (124M), Qwen-2.5-0.5B, TinyLlama-1.1B, Pythia-1.4B, and Phi-3-mini-4k-instruct (3.8B).
•	Standard Closure Configuration (EXP15–EXP16): 
o	Base Model: microsoft/Phi-3-mini-4k-instruct (LoRA targets: qkv_proj, o_proj, r=8r=8, α=16α=16).
o	Hyperparameters: 8 epochs, 60 samples per epoch, deterministic fixed seeds (SEEDS = [11, 22, 33]).
o	Validation: Token-weighted perplexity over an extended corpus of 40 multi-domain neutral sentences.
________________________________________
5. Key Results (EXP15 – EXP16)
5.1. Surgical Ablation (EXP15)
Three training branches were compared under identical conditions: NONE (no gate or contrast), GATE_ONLY (active filter, β=0β=0), and BEATRIZ (active filter with contrastive term, β=1.0β=1.0).
Branch	Training Margin	Held-Out Margin (n=2)	Perplexity (PPL)
BASE (untrained)	+1.34+1.34	+1.90+1.90	12.712.7
NONE	−0.03±0.02−0.03±0.02	+3.57±0.17+3.57±0.17	30.930.9
GATE_ONLY	+7.46±0.24+7.46±0.24	+5.08±0.09+5.08±0.09	58.858.8
BEATRIZ	+10.13±0.07+10.13±0.07	+5.91±0.07+5.91±0.07	86.386.3
Effect breakdown on held-out: Pure filtering (GATE_ONLY) provides 65% of the total defensive benefit; the contrastive term (BEATRIZ) adds the remaining 35%, albeit at a perplexity cost 1.8 times higher per unit of margin. This validates a practical recommendation for startups: GATE_ONLY is the default operating point (it does not require modifying the student loop and saturates in 2 epochs); BEATRIZ is reserved for scenarios demanding maximum epistemic purity at the expense of a higher PPL cost.
5.2. Verified Bit-Exact Determinism
Independent runs executed on different days under the same seed protocol yielded bit-identical results across all epochs and variables (identical SHA-256 hashes on LoRA adapters). This demonstrates that the experimental protocol is rigorously reproducible, a rare guarantee in LLM research.
5.3. Scaled Held-Out and Gate Confusion Matrix (EXP16)
To eradicate the initial statistical weakness (n=2n=2 facts in held-out), EXP16 scaled the benchmark to 30 multi-domain unattacked facts (physics, medicine, history, biology, computer science) and integrated a strict log of the filter's confusion matrix.
•	Scaled Held-Out (n=30n=30): The positive margin consolidates at +4.19±0.08+4.19±0.08 (BEATRIZ), conclusively proving that the improvement in unattacked facts is a systemic phenomenon of fine-tuning guided by fluent and anchored prose, not a statistical anomaly.
•	Gate Performance (Real Confusion Matrix): 
o	Precision: 0.930.93
o	Recall: 0.800.80
o	Interpretation: The filter succeeds in 93 out of every 100 blocks of falsehoods in the stream, catching 80% of malicious attempts without collapsing due to massive false positives.
o	Logic Latency: ∼0.1 ms∼0.1 ms per decision on a cached path (∼10,000∼10,000 decisions/second per thread).
________________________________________
6. Declared Limitations
With scientific honesty, the following limits are recognized in the current state of the project:
1.	Anchor Corpus Size: The current corpus consists of 36 total elements (6 training, 30 held-out). Scaling to hundreds of anchors is necessary to evaluate behavior in massive open domains.
2.	Live Path Latency: The 0.1 ms0.1 ms measurement corresponds to logic in cached memory; in production deployment, the embedding of the candidate text depends on a forward pass oracle inference step, suggesting the future need for a specialized lightweight encoder (e.g., E5-small).
3.	Commercial Licensing: The current code operates under the PolyForm Noncommercial 1.0.0 license for auditing, research, and defense purposes, requiring specific agreements for direct commercial exploitation.
________________________________________
7. Conclusion: Scientific Sovereignty is Possible
Beatriz's experimental series demonstrates that security and governance of language models are not the monopoly of large technological conglomerates. Through a clean architectural design—a non-invasive epistemic proxy coupled to a verified anchor corpus—it is possible to neutralize surgical data poisoning attacks with modest resources.
We invite the community of independent developers, academic researchers, and small startups to replicate these experiments using the code available in the official repository, as the rest of the series will be periodically published for their use, proving that true innovation in AI stems from freedom, rigor, and open source.
