# EdgeGuard-NS

**Learned Neurosymbolic Fusion for Multi-Attack and Zero-Day Detection in Resource-Constrained IIoT**

EdgeGuard-NS combines a compact neural network (~50k parameters, ~200 KB FP32 / smaller INT8) with a small set of human-auditable traffic rules whose influence is *learned from data*. The result is an intrusion detection model that is accurate on known attacks, measurably better at flagging never-seen attack families, fully interpretable at the rule level, and small and fast enough to deploy on a single CPU thread at the edge.

Everything - preprocessing, training, fusion, baselines, statistics, figures, and artifact export - lives in one reproducible notebook: [`EdgeguardNS.ipynb`](EdgeguardNS.ipynb).

---

## Key ideas

### 1. Entropy-gated Learned Symbolic Evidence Fusion (LSEF)

Fixed, auditable rules (flood-rate thresholds, half-open SYN probes, service-port checks, RST/ICMP logic) produce per-flow activations. Each rule carries a **learnable evidence weight**, initialized at **0** — i.e., the model starts as the exact neural baseline. Weights are trained on a validation split and kept **only if they improve held-out macro-F1** (a do-no-harm gate).

The symbolic bias is scaled by the network's **normalized predictive entropy**:

'''
logits_final = logits + H(x) · (A(x) ⊙ w) · M
'''

Confident predictions ('H ≈ 0') are never perturbed; rules only reshape decisions the network is already unsure about. Rule thresholds are percentiles of *benign training traffic only* — no test or label leakage. Learned weights are reported as an interpretability table (Figure 8).

### 2. Zero-day (open-set) detection with negative controls

An entire attack family is withheld from training and must be detected at test time. Symbolic evidence acts as an **additive channel on top of any base OOD score** (reported for MSP and energy), with weights that are:

- **Non-negative and benign-grounded** - a rule's weight is its rarity in benign traffic, so a firing rule can never count *against* detecting an unseen family;
- **Bounded in [0, 1]** - validation-calibrated thresholds transfer stably;
- **Double-guarded** - the fusion strength λ is the largest grid value that degrades *neither* validation AUROC *nor* validation recall at the 95%-TNR operating point.

Each negative-control holdout is validated by a rule-activation diagnostic (weighted activation must stay below 5%), and the method is compared against **MSP, ODIN, and Mahalanobis** under identical models and splits. A coverage dose–response analysis (Figure 6b) shows the AUROC gain appears when — and only when — the withheld family actually exhibits the rule-encoded behaviors.

### 3. Baselines under identical splits

Random Forest, HistGradientBoosting, and a compact 1-D CNN (TabCNN) are trained on the same splits and profiled for accuracy, macro-F1, **single-flow latency on one CPU thread**, and serialized size — against the BN-folded FP32 and dynamically quantized INT8 variants of EdgeGuard-NS.

### 4. Statistical rigor

- **10-seed protocol** (fresh split + init per seed), mean ± std everywhere
- Paired **Wilcoxon** and **t-tests**, plus **bootstrap 95% CIs** and **Cohen's d_z**
- Macro-F1-first reporting for imbalanced classes
- Leak-free preprocessing (identifiers/IPs dropped, dedup, scaler fit on train only)
- Optional **temporal (chronological) split** for Edge-IIoTset so flows from one capture session cannot straddle train and test
- INT8 deployment with numerical verification of BN-folding and an accuracy-safety fallback to FP32

## Datasets

| Dataset | Label column | Notes |
|---|---|---|
| [RT-IoT2022](https://www.kaggle.com/datasets/supplejade/rt-iot2022real-time-internet-of-things) | 'Attack_type' | Used as-is |
| [NF-ToN-IoT-v2](https://www.kaggle.com/datasets/dhoogla/nftoniotv2) | 'Attack' | 16.9M rows; stratified per-class cap of 100k (auditable — full vs capped counts printed) |
| [Edge-IIoTset](https://www.kaggle.com/datasets/mohamedamineferrag/edgeiiotset-cyber-security-dataset-of-iot-iiot) | 'Attack_type' | Supports per-class chronological 80/20 split via 'frame.time` |

Datasets are **not** included in this repository. Download them (e.g., via Kaggle/kagglehub) and point the notebook at your data root:

'''bash
export EDGEGUARD_DATA=/path/to/datasets
'''

The expected layout under that root matches the kagglehub download structure (see Cell 2 of the notebook).

## Getting started

'''bash
pip install numpy pandas matplotlib scipy scikit-learn torch pyarrow jupyter
jupyter notebook EdgeguardNS.ipynb
'''

Run the cells top to bottom. Key configuration knobs live in **Cell 1**:

| Knob | Default | Meaning |
|---|---|---|
| 'N_SEEDS' | 10 | Seeds for the robustness protocol |
| 'TEMPORAL_SPLIT_EDGE' | 'True' | Chronological split for Edge-IIoTset |
| 'SINGLE_THREAD_LATENCY' | 'True' | "Edge-honest" latency profiling on one CPU thread |
| 'NF_MAX_PER_CLASS' | 100,000 | Per-class cap for NF-ToN-IoT-v2 ('None' = all rows) |
| 'EPOCHS' / 'PATIENCE' | 120 / 12 | Training budget with early stopping |

A GPU speeds up training but is not required.

## Notebook structure

| Cells | What they do |
|---|---|
| 1–3 | Config, data loading, leak-free preprocessing (closed-set, open-set, temporal) |
| 4 | 'EdgeGuardNet', 'SymbolicRuleEngine', entropy-gated 'LearnedFusion', BN-folding utilities |
| 5–6 | Fixed-seed reference run + 10-seed robustness protocol |
| 7–9 | Training curves, aggregated metrics + significance tests, confusion matrices, per-class F1 |
| 10 | Classical + deep baselines, deployment profiling (latency / size / INT8) |
| 11–11b | Zero-day experiment with negative controls, OOD baselines, coverage dose–response |
| 12–13 | Seed stability, learned rule weights, summary tables, confusion-derived rates, artifact export |

## Outputs

Running the notebook produces:

- **Figures** ('figures/'): class distributions, training dynamics, confusion matrices, per-class F1, baseline comparison, zero-day results, coverage dose–response, seed stability, learned rule weights, detection rates
- **CSVs**: 'edgeguard_ns_summary.csv', 'edgeguard_ns_per_seed.csv', 'baseline_comparison.csv', 'zero_day_results.csv', 'perclass_confusion_rates.csv', 'binary_detection_rates.csv'
- **Models**: 'edgeguard_ns_<dataset>.pt' - state dict, learned fusion weights, rule names, class labels, and feature names for each dataset

All results are reported as measured; no target-driven adjustment is applied.

## Citation
If you use this code, please cite this repository (and the dataset papers for RT-IoT2022, NF-ToN-IoT-v2, and Edge-IIoTset).


## License
No license file is included yet - add one (e.g., MIT or Apache-2.0) before publishing.
