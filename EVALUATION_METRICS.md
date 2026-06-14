# Evaluation Metrics Guide for `injection_sol`

> **Purpose:** This document identifies which research paper is most aligned with this
> system, explains why, and defines a complete set of evaluation metrics to use —
> grounded in published benchmarks so results are directly comparable to the literature.

---

## 1. Paper Alignment Analysis

Three papers were reviewed:

| # | ArXiv ID | Title | Venue |
|---|----------|-------|-------|
| A | 2312.14197 | *Benchmarking and Defending Against Indirect Prompt Injection Attacks on Large Language Models* | KDD 2025 |
| B | 2505.06311 | *Defending against Indirect Prompt Injection by Instruction Detection* (InstructDetector) | 2025 |
| C | 2310.12815 | *Formalizing and Benchmarking Prompt Injection Attacks and Defenses* | USENIX Security 2024 |

### Recommendation: **Paper A (2312.14197) as primary, Paper B (2505.06311) as secondary**

#### Why Paper A is the primary reference

Paper A introduces **BIPIA** — the exact benchmark already named in this project's
`design.md`, `DATASETS.md`, and `docs/REPORT.md`. The alignment is direct:

| Dimension | Paper A (BIPIA) | This system (`injection_sol`) |
|-----------|-----------------|-------------------------------|
| Threat model | Indirect prompt injection via external documents | Same — document-borne payloads only |
| Defense targets | RAG-style LLM pipelines reading external content | RAG pipeline with ingest + retrieval |
| Primary metric | Attack Success Rate (ASR) | ASR (defended vs. undefended) |
| Task coverage | Email QA, Web QA, Table QA, Summarization, Code QA | Uses BIPIA as evaluation corpus |
| Attack positions | Beginning / middle / end of document | Chunk-level injection |
| Defense types | Black-box (prompting) + White-box (fine-tuning) | ML detector (D1 + D2) — new class |

Paper A also provides concrete ASR baselines for 25+ LLMs (GPT-4 = 0.31, GPT-3.5 = 0.26)
that make the "undefended ASR" column in the `compare_defense()` harness directly
benchmarkable against published numbers.

#### Why Paper B is the secondary reference

Paper B (InstructDetector) is the closest architectural cousin: both papers and this system
try to **detect** injection rather than merely prompt the model to resist it. InstructDetector
reports **in-domain / out-of-domain accuracy splits** and a cross-LLM ASR comparison table
that map directly onto the `evaluate_detection()` + `cross_llm_asr()` functions already in
`src/injection_sol/eval/harness.py`.

#### Why Paper C is tertiary

Paper C (USENIX 2024) covers both direct and indirect injection and is primarily a
benchmarking/taxonomy framework. It is useful for understanding the attack landscape but its
metrics are not as directly applicable to a detection-focused RAG defense system.

---

## 2. Metric Categories

```
Group 1 — Chunk-Level Detection           (D1, D2, combined)
Group 2 — Defense Effectiveness           (ASR comparison)
Group 3 — Output Quality Preservation     (utility after defense)
Group 4 — Ablation / Per-Gate Analysis    (D1-only vs D2-only vs FULL)
Group 5 — Robustness / Generalization     (in-domain vs out-of-domain, attack positions)
Group 6 — Operational / Deployment        (latency, threshold tunability)
```

---

## 3. Group 1 — Chunk-Level Detection Metrics

These measure how accurately the two-gate defense identifies injected chunks.
They are computed by `eval/metrics.py` and `eval/harness.py::evaluate_detection()`.

### 3.1 Accuracy

```
Accuracy = (TP + TN) / (TP + TN + FP + FN)
```

- **What it captures:** Overall correctness of the binary classifier.
- **Paper A baseline:** Not directly reported (ASR is the primary metric there).
- **Paper B baseline:** 99.60% in-domain, 96.90% out-of-domain.
- **Target for this system:** ≥ 95% out-of-domain (mirroring InstructDetector OOD).
- **Code:** `metrics_at_threshold(y_true, scores, threshold)["accuracy"]`

### 3.2 Precision

```
Precision = TP / (TP + FP)
```

- **What it captures:** Of all chunks flagged as injected, how many truly are.
  High precision means few legitimate chunks are falsely blocked.
- **Importance:** Low precision causes over-blocking and degrades answer quality.
- **Target:** ≥ 0.90 at the BLOCKED threshold.
- **Code:** `metrics_at_threshold(...)["precision"]`

### 3.3 Recall (Sensitivity / True Positive Rate)

```
Recall = TP / (TP + FN)
```

- **What it captures:** Of all truly injected chunks, how many the defense catches.
  High recall means fewer attacks slip through.
- **Importance:** The primary security guarantee — missed injections reach the LLM.
- **Target:** ≥ 0.90 at the BLOCKED threshold; push toward 0.99 with threshold tuning.
- **Code:** `metrics_at_threshold(...)["recall"]`

### 3.4 F1-Score

```
F1 = 2 × (Precision × Recall) / (Precision + Recall)
```

- **What it captures:** Harmonic mean balancing precision and recall.
- **Importance:** Single-number summary when precision/recall trade-off matters.
- **Target:** ≥ 0.90.
- **Code:** `metrics_at_threshold(...)["f1"]`

### 3.5 False Positive Rate (FPR)

```
FPR = FP / (FP + TN)
```

- **What it captures:** Fraction of clean chunks wrongly flagged as injected.
  This directly governs usability — high FPR means legitimate content is blocked.
- **Importance:** Paper A shows that defenses with high FPR are impractical even if ASR is
  low. Paper B (InstructDetector) achieves near-zero FPR as a key selling point.
- **Target:** ≤ 0.05 (5% of benign chunks falsely blocked) at the BLOCKED threshold.
- **Code:** `metrics_at_threshold(...)["fpr"]`; also `evaluate_detection()["clean_false_positives"]`

### 3.6 ROC Curve and AUC (Area Under the Curve)

- **What it captures:** The full precision–recall (or TPR–FPR) trade-off across all possible
  thresholds. AUC = 1.0 is a perfect detector; 0.5 is random.
- **Importance:** Threshold-independent measure of detector quality; required for comparing
  D1-only, D2-only, and combined configurations at a single glance.
- **Target:** AUC ≥ 0.95 for the combined risk score.
- **Current result (from `data/eval/detection_report.json`):** AUC = **0.875** on 5-sample
  pilot — needs re-evaluation on a full BIPIA-sized set.
- **Code:** `metrics.roc_curve(y_true, scores)`; `evaluate_detection()["combined_roc"]["auc"]`

### 3.7 Threshold Sweep

- Run metrics at every integer threshold from 0 to 100 (step = 1).
- Plot Precision, Recall, F1, FPR vs. threshold to find the optimal operating point.
- Used to justify the chosen `RISK_THRESHOLD_BLOCK` and `RISK_THRESHOLD_SAFE` values.
- **Code:** `metrics.threshold_sweep(y_true, scores)`

---

## 4. Group 2 — Defense Effectiveness (ASR)

The **Attack Success Rate (ASR)** is the central security metric from Paper A (BIPIA)
and Paper B (InstructDetector). It measures how often an attacker succeeds
end-to-end — i.e., the LLM follows the hidden instruction despite the defense.

```
ASR = (number of queries where injected instruction was followed)
      / (total number of attack scenarios)
```

### 4.1 Undefended ASR (`DEFENSE_VARIANT = NONE`)

- Baseline: what fraction of attacks succeed with no defense active.
- **Paper A benchmarks (GPT-4 undefended):** 0.31; GPT-3.5: 0.26; WizardLM-70B: 0.08.
- This number should be **high** — confirming the threat is real on the chosen LLM.
- **Code:** `run_asr(scenarios, defense_variant=DefenseVariant.NONE)`

### 4.2 Defended ASR (`DEFENSE_VARIANT = FULL`)

- Same scenarios, defense active (D1 + D2 + risk scoring + response policy).
- **Paper B benchmark (InstructDetector):** 0.03%–0.13% across GPT-3.5/4/Vicuna.
- **Paper A white-box defense:** 0.0047–0.0053 (Vicuna fine-tuned on GPT-4 responses).
- **Target for this system:** ≤ 0.10 (10%) after defense.
- **Code:** `run_asr(scenarios, defense_variant=DefenseVariant.FULL)`

### 4.3 Absolute ASR Reduction

```
ΔasrAbs = ASR_undefended − ASR_defended
```

- Shows the raw benefit of the defense in percentage points.
- **Target:** ≥ 0.20 (20 pp reduction from an assumed 30%+ undefended rate).
- **Code:** `compare_defense()["absolute_reduction"]`

### 4.4 Relative ASR Reduction

```
ΔasrRel = ΔasrAbs / ASR_undefended
```

- Normalizes the reduction by how severe the undefended threat is.
- **Target:** ≥ 0.70 (70% of attacks blocked).
- **Code:** `compare_defense()["relative_reduction"]`

### 4.5 Cross-LLM ASR Table

Evaluate undefended vs. defended ASR across at minimum three models:

| Model | ASR Undefended | ASR Defended | ΔasrAbs | ΔasrRel |
|-------|----------------|--------------|---------|---------|
| GPT (API) | — | — | — | — |
| LLaMA (local quantized) | — | — | — | — |
| Mistral (local quantized) | — | — | — | — |

- **Paper A** reports cross-model vulnerability confirming that no LLM is immune.
- **Paper B** reports per-model defended ASR for GPT-3.5, GPT-4, Vicuna, Qwen.
- **Code:** `cross_llm_asr(scenarios, models=["gpt-...", "llama-...", "mistral-..."])`

---

## 5. Group 3 — Output Quality Preservation

A defense that blocks too aggressively degrades the RAG answer quality even on
legitimate queries. These metrics capture the utility cost of the defense.

### 5.1 ROUGE-1 (Recall)

```
ROUGE-1 = overlap of unigrams between defended answer and reference answer
```

- **Paper A baseline:** 0.62–0.68 range on clean scenarios.
- Measures whether legitimate information is still present in the answer after
  the defense masks or drops chunks.
- **Target:** ROUGE-1 ≥ 0.60 on benign queries (no injections) when defense is active.
- **How to compute:** Run the full RAG pipeline with `DEFENSE_VARIANT=FULL` on a benign
  document set, compare answers to reference answers using `rouge-score` library.

### 5.2 Warning Answer Rate

```
WarningRate = (queries that produced a warning answer) / (total clean queries)
```

- System-specific metric: when all retrieved chunks are BLOCKED, the system emits a
  warning answer instead of a generated one (`result.is_warning == True`).
- On **clean** (non-injected) queries this should be near zero; any non-zero value
  means legitimate content is being over-blocked.
- **Target:** ≤ 0.02 (≤ 2% of clean queries trigger a warning).
- **Code:** check `result.is_warning` in `ASRResult.scenarios`

### 5.3 Chunks Blocked on Clean Queries (False Block Rate)

```
FalseBlockRate = (chunks blocked by BLOCKED decision on benign docs)
                 / (total chunks retrieved on benign queries)
```

- Companion to FPR at the chunk level but measured end-to-end through the pipeline.
- **Target:** ≤ 0.05.
- **Code:** `evaluate_detection()["clean_false_positives"]` / `n_negative`

---

## 6. Group 4 — Ablation / Per-Gate Analysis

These metrics isolate the contribution of D1 and D2 individually, and justify the
two-gate design.

### 6.1 D1-Only vs D2-Only vs FULL

Run the same scenario set under three `DEFENSE_VARIANT` settings and compare:

| Defense Mode | ASR | Recall | FPR | AUC |
|---|---|---|---|---|
| NONE | (baseline) | — | — | — |
| D1 | — | — | — | — |
| D2 | — | — | — | — |
| FULL | — | — | — | — |

- Demonstrates **complementarity**: attacks that bypass D1 (novel phrasing) should be
  caught by D2 (embedding geometry), and vice versa.
- **Code:** set `DEFENSE_VARIANT` in `.env` to `D1`, `D2`, `FULL`; run `run_eval.py` each time.

### 6.2 D1 Metrics at Block Threshold

Already computed in `evaluate_detection()["d1_at_block_threshold"]`:
- Accuracy, Precision, Recall, F1, FPR — report all five.

### 6.3 Combined Risk Score Metrics at Block Threshold

Already computed in `evaluate_detection()["combined_at_block_threshold"]`:
- Same five metrics as above, for the fused D1 + D2 signal.
- The delta between D1-only and Combined is the measurable contribution of D2.

---

## 7. Group 5 — Robustness and Generalization

These metrics verify the defense works beyond the training distribution,
motivated by Paper B's in-domain / out-of-domain gap and Paper A's task/position analysis.

### 7.1 In-Domain vs Out-of-Domain Detection Accuracy

- **In-domain:** test on examples from the same distribution as D1 training data.
- **Out-of-domain:** test on BIPIA or a held-out corpus with different phrasing/topics.
- **Paper B target gap:** ≤ 3 pp drop (99.60% ID → 96.90% OOD).
- Report both; a large gap indicates D1 overfits to training attack phrasing.

### 7.2 ASR per Task Type (from BIPIA)

Following Paper A's five-scenario breakdown:

| Task | ASR Undefended | ASR Defended |
|------|----------------|--------------|
| Email QA | — | — |
| Web QA | — | — |
| Table QA | — | — |
| Summarization | — | — |
| Code QA | — | — |

- Identifies which RAG use-cases are harder to defend.
- **Paper A finding:** end-of-content attacks tend to have higher ASR — verify if
  the same holds here.

### 7.3 ASR per Attack Position

| Injection position | ASR Undefended | ASR Defended |
|--------------------|----------------|--------------|
| Beginning of chunk | — | — |
| Middle of chunk | — | — |
| End of chunk | — | — |

- Paper A found end-of-content attacks achieve highest ASR.
- Tests whether D1's pattern matching and D2's geometry signal are position-agnostic.

### 7.4 ASR on Obfuscated / Paraphrased Attacks

- Generate a small set of paraphrased injection payloads (synonym substitution,
  unicode homoglyphs, mixed-language) and measure defended ASR.
- **Purpose:** quantifies the brittleness gap that motivated D2 (unsupervised) alongside D1.
- **Target:** defended ASR on obfuscated attacks ≤ 2× defended ASR on standard attacks.

---

## 8. Group 6 — Operational Metrics

### 8.1 Per-Chunk Scoring Latency (ms)

- D1 inference time per chunk (CPU, DeBERTa-v3-small).
- D2 scoring time per retrieved set.
- Total defense overhead per query.
- **Practical constraint:** the system runs on i7 / 8 GB RAM; document this baseline.

### 8.2 Threshold Sensitivity (Stability)

- Vary `RISK_THRESHOLD_BLOCK` ± 10 points and report how much ASR and FPR change.
- Demonstrates the calibrated score is smooth enough for operational tuning.

---

## 9. Summary Table — What to Report

| # | Metric | Group | Code Location | Paper Source |
|---|--------|-------|---------------|--------------|
| 1 | Accuracy (combined, at block threshold) | Detection | `metrics_at_threshold` | Paper B |
| 2 | Precision | Detection | `metrics_at_threshold` | Paper B |
| 3 | Recall / TPR | Detection | `metrics_at_threshold` | Papers A, B |
| 4 | F1-Score | Detection | `metrics_at_threshold` | Paper B |
| 5 | FPR | Detection | `metrics_at_threshold` | Paper B |
| 6 | ROC / AUC | Detection | `roc_curve` | Paper B |
| 7 | Threshold sweep plot | Detection | `threshold_sweep` | Papers A, B |
| 8 | ASR (undefended) | Effectiveness | `run_asr(NONE)` | Papers A, B, C |
| 9 | ASR (defended, FULL) | Effectiveness | `run_asr(FULL)` | Papers A, B |
| 10 | Absolute ASR reduction | Effectiveness | `compare_defense` | Paper A |
| 11 | Relative ASR reduction | Effectiveness | `compare_defense` | Paper A |
| 12 | Cross-LLM ASR table | Effectiveness | `cross_llm_asr` | Papers A, B |
| 13 | ROUGE-1 (clean queries) | Quality | external lib | Paper A |
| 14 | Warning answer rate | Quality | `result.is_warning` | System-specific |
| 15 | Chunk false-block rate | Quality | `clean_false_positives` | Paper B |
| 16 | D1-only / D2-only / FULL ablation | Ablation | `DEFENSE_VARIANT` | System-specific |
| 17 | In-domain vs OOD accuracy | Robustness | `evaluate_detection` | Paper B |
| 18 | ASR per BIPIA task type | Robustness | `run_asr` per scenario | Paper A |
| 19 | ASR per injection position | Robustness | `run_asr` per scenario | Paper A |
| 20 | ASR on obfuscated attacks | Robustness | custom scenario set | Paper B |

---

## 10. Published Benchmarks to Compare Against

Use these numbers to contextualize your results:

### Undefended ASR (Paper A — BIPIA, no defense)
| LLM | Overall ASR |
|-----|-------------|
| GPT-4 | 0.3103 |
| GPT-3.5-turbo | 0.2616 |
| WizardLM-70B | 0.0795 |
| Vicuna-13B | 0.1294 |
| Average (25 models) | 0.1179 |

### Defended ASR (Paper A — BIPIA, white-box fine-tuning)
| LLM | Defended ASR |
|-----|--------------|
| Vicuna-7B (fine-tuned on GPT-4) | 0.0053 |
| Vicuna-13B (fine-tuned on GPT-4) | 0.0047 |

### Detection Accuracy (Paper B — InstructDetector)
| Setting | Accuracy |
|---------|----------|
| In-domain (Wikipedia + LaMini) | 99.60% |
| Out-of-domain (News + BIPIA) | 96.90% |
| OOD (Wiki + BIPIA) | 99.45% |

### ASR with InstructDetector active (Paper B)
| LLM | ASR |
|-----|-----|
| GPT-3.5-Turbo | 0.12% |
| GPT-4o | 0.13% |
| Vicuna-7B | 0.03% |
| Qwen2.5-7B | 0.10% |

---

## 11. Minimum Viable Evaluation (if time is limited)

If only partial evaluation is feasible, prioritize in this order:

1. **AUC** — one number that summarizes detector quality (Group 1).
2. **Precision, Recall, FPR at block threshold** — three numbers that summarize the
   operational trade-off (Group 1).
3. **Undefended ASR vs. Defended ASR** on one LLM — the core security claim (Group 2).
4. **D1-only vs. FULL** — proves D2 adds value (Group 4).
5. **Warning answer rate on clean queries** — proves usability is not destroyed (Group 3).

---

*Generated 2026-06-14. Based on:*
- *Paper A: Yi Zeng et al., "Benchmarking and Defending Against Indirect Prompt Injection Attacks on Large Language Models," KDD 2025 (arXiv:2312.14197)*
- *Paper B: Tongyu Wen et al., "Defending against Indirect Prompt Injection by Instruction Detection," arXiv:2505.06311*
- *Paper C: Jingwei Yi et al., "Formalizing and Benchmarking Prompt Injection Attacks and Defenses," USENIX Security 2024 (arXiv:2310.12815)*
