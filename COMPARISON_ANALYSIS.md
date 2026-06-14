# Research Paper Comparison Analysis for `injection_sol`

> **Purpose:** Analyze three candidate papers, select the most suitable one for benchmarking
> this system, and define the evaluation metrics to use for comparison.
>
> **System under evaluation:** `injection_sol` — a two-gate indirect prompt injection defense
> for RAG pipelines, combining a supervised classifier (D1: DeBERTa-v3-small) with an
> unsupervised embedding anomaly detector (D2: Isolation Forest) to produce a calibrated
> 0–100 risk score and a SAFE / SUSPICIOUS / BLOCKED response policy.

---

## 1. Overview of the Three Candidate Papers

| # | arXiv | Title | Venue | Year |
|---|-------|-------|-------|------|
| **A** | [2312.14197](https://arxiv.org/abs/2312.14197) | *Benchmarking and Defending Against Indirect Prompt Injection Attacks on Large Language Models* | **KDD 2025** | 2025 |
| **B** | [2505.06311](https://arxiv.org/abs/2505.06311) | *Defending against Indirect Prompt Injection by Instruction Detection* (InstructDetector) | arXiv | 2025 |
| **C** | [2310.12815](https://arxiv.org/abs/2310.12815) | *Formalizing and Benchmarking Prompt Injection Attacks and Defenses* | **USENIX Security 2024** | 2024 |

---

## 2. Paper-by-Paper Analysis

### Paper A — BIPIA (arXiv:2312.14197)

**Authors:** Jingwei Yi, Yueqi Xie, Bin Zhu, Emre Kiciman, Guangzhong Sun, Xing Xie, Fangzhao Wu (Microsoft Research)

**Core Contribution:**
Introduces **BIPIA** (Benchmark for Indirect Prompt Injection Attacks) — the first comprehensive benchmark specifically designed for indirect prompt injection in LLM-integrated applications. The paper evaluates 25+ LLMs and proposes two classes of defenses.

**Threat Model:**
- Attacker controls external documents (emails, web pages, tables, code, summaries)
- User makes a benign query; RAG retrieves the poisoned document
- The hidden instruction hijacks the LLM's response
- Five task types: Email QA, Web QA, Table QA, Summarization, Code QA

**Defenses Proposed:**
1. **Black-box defense:** Boundary awareness (XML tags separating data from instructions) + explicit reminder in system prompt
2. **White-box defense:** Fine-tune the LLM on a curated dataset to ignore injected instructions

**Key Reported Metrics:**
- Primary metric: **Attack Success Rate (ASR)** = fraction of scenarios where the LLM follows the injected instruction
- Secondary: Answer quality under clean conditions (ROUGE-based)
- Undefended ASR across 25 LLMs

**Selected Numerical Results:**
| LLM | Undefended ASR | White-box Defended ASR |
|-----|---------------|----------------------|
| GPT-4 | 31.03% | — |
| GPT-3.5-Turbo | 26.16% | — |
| Vicuna-13B | 12.94% | **0.47%** |
| WizardLM-70B | 7.95% | — |
| Average (25 models) | 11.79% | — |

**Limitation relative to this system:** Paper A defenses operate at the *LLM prompt level* (prompting + fine-tuning). They do not use a separate detection module. There is no precision/recall/F1 analysis because the defense is not a detector — it is a resistance mechanism built into the model's behavior.

---

### Paper B — InstructDetector (arXiv:2505.06311)

**Authors:** Tongyu Wen, Chenglong Wang, Xiyuan Yang, Haoyu Tang, Yueqi Xie, Lingjuan Lyu, Zhicheng Dou, Fangzhao Wu

**Core Contribution:**
Proposes **InstructDetector**, a detection-based defense that intercepts injected instructions *before* they reach the LLM. It analyzes hidden states and gradients from intermediate transformer layers to identify whether a retrieved chunk contains an instruction that would alter model behavior.

**Threat Model:**
- Identical to Paper A: external content (RAG) contains hidden directives
- Evaluated on the BIPIA benchmark (same dataset as Paper A)
- Cross-LLM evaluation: GPT-3.5-Turbo, GPT-4o, Vicuna-7B, Qwen2.5-7B

**Defense Mechanism:**
- Extracts intermediate-layer features (hidden states + gradients) while the LLM processes the retrieved chunk
- Trains a lightweight binary classifier on these features to detect "is this chunk an instruction?"
- If flagged: chunk is dropped from the RAG context before generation

**Key Reported Metrics:**
- Detection accuracy (in-domain and out-of-domain)
- ASR after defense activation (defended ASR across multiple LLMs)

**Selected Numerical Results:**

| Setting | Accuracy |
|---------|----------|
| In-domain (Wikipedia + LaMini) | **99.60%** |
| Out-of-domain (News + BIPIA) | **96.90%** |
| OOD alternative (Wiki + BIPIA) | **99.45%** |

| LLM | Undefended ASR | InstructDetector ASR |
|-----|---------------|----------------------|
| GPT-3.5-Turbo | ~26% | **0.12%** |
| GPT-4o | ~31% | **0.13%** |
| Vicuna-7B | ~13% | **0.03%** |
| Qwen2.5-7B | ~11% | **0.10%** |

**Strength relative to this system:** InstructDetector is the closest *architectural cousin* — both are detection-based defenses. It reports both detection metrics and ASR metrics, making direct comparison possible. However, InstructDetector is **white-box** (requires access to LLM internal layers), while `injection_sol` is **black-box compatible**.

---

### Paper C — Open-Prompt-Injection (arXiv:2310.12815)

**Authors:** Yupei Liu, Yuqi Jia, Runpeng Geng, Jinyuan Jia, Neil Zhenqiang Gong (USENIX Security 2024)

**Core Contribution:**
Provides a **formal taxonomy** of prompt injection attacks and evaluates 5 attacks × 10 defenses × 10 LLMs × 7 tasks. Releases the Open-Prompt-Injection benchmarking platform.

**Threat Model:**
- Covers **both** direct injection (attacker controls user input) and indirect injection (attacker controls external data)
- Broader scope than Papers A and B

**Defenses Evaluated:**
Input/output filtering, instruction-based protections (sandwich defense, data isolation), and model-level mitigations.

**Limitation relative to this system:**
- Not focused exclusively on RAG or indirect injection
- Does not report detailed per-defense detection metrics (precision, recall, AUC)
- The benchmark is primarily a taxonomy and evaluation platform, not a defense proposal
- No specific numeric results comparable to D1/D2 gate performance

---

## 3. Paper Selection Recommendation

### Primary: **Paper B — InstructDetector (arXiv:2505.06311)**

**Rationale:**

| Comparison Dimension | InstructDetector (Paper B) | `injection_sol` |
|----------------------|---------------------------|-----------------|
| Defense class | Detection-based | Detection-based (D1 + D2) |
| Threat model | Indirect prompt injection via RAG | Same |
| Evaluation benchmark | BIPIA | BIPIA |
| Primary metrics | Detection accuracy + ASR | Detection accuracy + ASR |
| Cross-LLM evaluation | GPT-3.5, GPT-4o, Vicuna, Qwen | GPT, LLaMA, Mistral |
| Reports precision/recall? | Yes (in-domain + OOD) | Yes (via `eval/harness.py`) |
| Reports defended ASR? | Yes (per LLM) | Yes (via `run_asr()`) |
| Architecture access | **White-box** (needs LLM internals) | **Black-box** (external classifier) |

The key differentiator for comparison: `injection_sol` achieves detection **without access to LLM internal states**, making it deployable with any API-accessed LLM (GPT-4o via API, for instance). This is the central architectural claim to validate.

### Secondary: **Paper A — BIPIA (arXiv:2312.14197)**

Paper A provides the **undefended ASR baselines** (e.g., GPT-4 = 31%, GPT-3.5 = 26%) that establish what the threat looks like without any defense. These numbers anchor the "undefended" column in every ASR comparison table. They also define the five BIPIA task categories used to break down per-task ASR.

### Role of Paper C

Paper C is useful as **background context** for the taxonomy of attacks, but its metrics are not directly comparable to a RAG-specific detector. Do not use it as the primary comparison target.

---

## 4. Metrics to Evaluate

Based on the above analysis, the following metrics should be measured and compared against Paper B's reported values (with Paper A providing ASR baselines).

### Group 1 — Detection Metrics (chunk-level)

These measure how accurately the two-gate system identifies injected chunks, directly comparable to InstructDetector's accuracy numbers.

| Metric | Formula | Target | Paper B Baseline | Why It Matters |
|--------|---------|--------|-----------------|---------------|
| **Accuracy** | (TP + TN) / N | ≥ 95% | 96.90% OOD | Overall correctness of binary classification |
| **Precision** | TP / (TP + FP) | ≥ 0.90 | Not separately reported | Low precision = legitimate chunks blocked |
| **Recall (TPR)** | TP / (TP + FN) | ≥ 0.90 | ≈ 99.60% (ID) | Low recall = attacks slip through to LLM |
| **F1-Score** | 2·P·R / (P+R) | ≥ 0.90 | Not separately reported | Balanced summary of P and R |
| **False Positive Rate (FPR)** | FP / (FP + TN) | ≤ 0.05 | Near-zero (implied) | Over-blocking degrades answer quality |
| **AUC-ROC** | Area under TPR–FPR curve | ≥ 0.95 | Not separately reported | Threshold-independent detector quality |

**In-domain vs Out-of-domain split** (motivated by Paper B's 99.60% vs 96.90% gap):
- Train D1 on one distribution; test on BIPIA as the OOD set
- Target: OOD accuracy drop ≤ 3 percentage points vs in-domain

**Threshold sweep:** plot Precision, Recall, F1, FPR against threshold 0–100 to justify the chosen `RISK_THRESHOLD_BLOCK` value.

---

### Group 2 — Defense Effectiveness (ASR)

The primary end-to-end security metric. Directly comparable to Paper A's undefended baselines and Paper B's defended ASR values.

```
ASR = (# scenarios where LLM followed the injected instruction) / (# total attack scenarios)
```

| Metric | Formula | Target | Reference |
|--------|---------|--------|-----------|
| **Undefended ASR** | `run_asr(NONE)` | Should match Paper A baseline (~26–31% for GPT) | Paper A |
| **Defended ASR (FULL)** | `run_asr(FULL)` | ≤ 10% | Paper B: 0.03–0.13% |
| **Absolute reduction** | ASR_none − ASR_full | ≥ 20 pp | Derived |
| **Relative reduction** | ΔAbs / ASR_none | ≥ 70% | Derived |

**Cross-LLM ASR table** (maps directly onto Paper B's per-model results):

| LLM | ASR Undefended | ASR Defended (FULL) | Absolute Reduction |
|-----|---------------|---------------------|--------------------|
| GPT (API) | (measure) | (measure) | (derive) |
| LLaMA-3.1-8B (local) | (measure) | (measure) | (derive) |
| Mistral-7B (local) | (measure) | (measure) | (derive) |

Paper B comparison rows:

| LLM | InstructDetector ASR |
|-----|---------------------|
| GPT-3.5-Turbo | 0.12% |
| GPT-4o | 0.13% |
| Vicuna-7B | 0.03% |
| Qwen2.5-7B | 0.10% |

> Note: `injection_sol` is not expected to match InstructDetector's near-zero ASR — InstructDetector has white-box access. The comparison is meant to quantify the **tradeoff** between access requirements and mitigation strength.

---

### Group 3 — Output Quality Preservation

A defense that over-blocks degrades the system for legitimate users. These metrics verify the utility cost is acceptable.

| Metric | Definition | Target | Reference |
|--------|-----------|--------|-----------|
| **ROUGE-1** | Unigram overlap between defended answer and reference, on clean (non-injected) queries | ≥ 0.60 | Paper A reports 0.62–0.68 on clean scenarios |
| **Warning Answer Rate** | % of clean queries where all chunks were blocked and the system returned a warning | ≤ 2% | System-specific |
| **False Block Rate** | % of benign chunks dropped on clean queries | ≤ 5% | Derived from FPR |

---

### Group 4 — Ablation (Per-Gate Contribution)

Justifies the two-gate design by showing D1 and D2 complement each other.

Run the same scenario set under each `DEFENSE_VARIANT` setting:

| Mode | ASR | Recall | FPR | AUC |
|------|-----|--------|-----|-----|
| NONE | (baseline) | — | — | — |
| D1 only | — | — | — | — |
| D2 only | — | — | — | — |
| FULL (D1 + D2) | — | — | — | — |

- **Expected finding:** D1 catches overt phrasing attacks; D2 catches evasive/paraphrased attacks where phrasing is disguised. FULL should outperform either gate alone.
- **Comparability to Paper B:** InstructDetector is architecturally a single gate (intermediate-layer features). The two-gate design is this system's novel contribution.

---

### Group 5 — Robustness and Generalization

Motivated by Paper A's task/position analysis and Paper B's in-domain vs OOD analysis.

**5a. ASR per BIPIA task type** (Paper A framework):

| Task | ASR Undefended | ASR Defended | Notes |
|------|---------------|--------------|-------|
| Email QA | — | — | |
| Web QA | — | — | |
| Table QA | — | — | |
| Summarization | — | — | |
| Code QA | — | — | |

**5b. ASR per injection position** (Paper A finding: end-of-content attacks have highest ASR):

| Position | ASR Undefended | ASR Defended |
|----------|---------------|--------------|
| Beginning of chunk | — | — |
| Middle of chunk | — | — |
| End of chunk | — | — |

**5c. ASR on obfuscated attacks** (Paper B motivation for OOD evaluation):
- Generate paraphrased/synonym-substituted payloads and measure defended ASR
- Target: defended ASR on obfuscated ≤ 2× defended ASR on standard payloads

---

### Group 6 — Operational Metrics

| Metric | Measurement | Notes |
|--------|------------|-------|
| **D1 inference latency** | ms per chunk (CPU, DeBERTa-v3-small) | Relevant for production feasibility |
| **D2 scoring latency** | ms per retrieved set | Includes anomaly scoring |
| **Total defense overhead** | ms added per query vs. undefended pipeline | Should be under 500ms for interactive use |
| **Threshold sensitivity** | ΔASR / ΔFPR when `RISK_THRESHOLD_BLOCK` ± 10 | Demonstrates smooth calibration |

---

## 5. Summary: All Metrics at a Glance

| # | Metric | Group | Comparable To |
|---|--------|-------|--------------|
| 1 | Accuracy (combined, at block threshold) | Detection | Paper B: 96.90% OOD |
| 2 | Precision | Detection | Paper B: implied high |
| 3 | Recall / TPR | Detection | Paper B: ≈ 99.60% ID |
| 4 | F1-Score | Detection | Paper B |
| 5 | FPR | Detection | Paper B: near-zero |
| 6 | AUC-ROC | Detection | Paper B |
| 7 | In-domain vs OOD accuracy gap | Detection | Paper B: 99.60% vs 96.90% |
| 8 | Threshold sweep (P/R/F1/FPR vs score) | Detection | — |
| 9 | Undefended ASR | Effectiveness | Paper A: 26–31% (GPT) |
| 10 | Defended ASR (FULL) | Effectiveness | Paper B: 0.03–0.13% |
| 11 | Absolute ASR reduction | Effectiveness | Paper A |
| 12 | Relative ASR reduction | Effectiveness | Paper A |
| 13 | Cross-LLM ASR table | Effectiveness | Paper B Table |
| 14 | ROUGE-1 on clean queries | Quality | Paper A: 0.62–0.68 |
| 15 | Warning answer rate | Quality | System-specific |
| 16 | False block rate on benign chunks | Quality | Paper B |
| 17 | D1-only vs D2-only vs FULL (ablation) | Ablation | System-specific |
| 18 | ASR per BIPIA task type | Robustness | Paper A |
| 19 | ASR per injection position | Robustness | Paper A |
| 20 | ASR on obfuscated payloads | Robustness | Paper B (OOD) |
| 21 | D1 inference latency (ms/chunk) | Operational | — |
| 22 | Total defense overhead (ms/query) | Operational | — |

---

## 6. Key Architectural Comparison

The table below positions `injection_sol` relative to Papers A and B on the most important architectural dimensions.

| Dimension | Paper A (BIPIA defenses) | Paper B (InstructDetector) | **injection_sol** |
|-----------|--------------------------|---------------------------|-------------------|
| Defense class | Prompting + Fine-tuning | Detection (white-box) | **Detection (black-box)** |
| LLM access required | Black-box / White-box | **White-box** (hidden states) | **Black-box only** |
| Detection stage | N/A (resistance, not detection) | Retrieval time | Ingestion (D1) + Retrieval (D2) |
| Number of gates | 1 (LLM behavior) | 1 (intermediate-layer classifier) | **2 (D1 + D2)** |
| Handles novel phrasing | Poor (prompting); Better (fine-tuned) | Good (feature-based) | D2 handles distribution shift |
| Deployable with API LLMs | Yes | No (needs layer access) | **Yes** |
| Evasion resistance | Moderate | Moderate | D2 provides unsupervised fallback |
| Published defended ASR | 0.47–0.53% (white-box) | 0.03–0.13% | Target: ≤ 10% |

---

## 7. Minimum Viable Evaluation (Priority Order)

If time is limited, report these in order:

1. **AUC** — threshold-independent detector quality (metric 6)
2. **Precision, Recall, FPR at block threshold** — operational trade-off (metrics 2–5)
3. **Undefended vs Defended ASR on one LLM** — core security claim (metrics 9–10)
4. **D1-only vs FULL ablation** — proves D2 adds measurable value (metric 17)
5. **ROUGE-1 and Warning Rate on clean queries** — proves usability is preserved (metrics 14–15)

---

## 8. Published Reference Numbers for Comparison Tables

Use these values to populate the comparison tables in the final report.

### Table R1 — Undefended ASR (Paper A, BIPIA benchmark, no defense)

| LLM | ASR |
|-----|-----|
| GPT-4 | 0.3103 (31.03%) |
| GPT-3.5-Turbo | 0.2616 (26.16%) |
| Vicuna-13B | 0.1294 (12.94%) |
| WizardLM-70B | 0.0795 (7.95%) |
| Average (25 models) | 0.1179 (11.79%) |

### Table R2 — Defended ASR (Paper A, white-box fine-tuning)

| Setup | Defended ASR |
|-------|-------------|
| Vicuna-7B fine-tuned on GPT-4 responses | 0.53% |
| Vicuna-13B fine-tuned on GPT-4 responses | 0.47% |

### Table R3 — Detection Accuracy (Paper B, InstructDetector)

| Split | Accuracy |
|-------|----------|
| In-domain (Wikipedia + LaMini) | 99.60% |
| Out-of-domain (News + BIPIA) | 96.90% |
| Out-of-domain (Wiki + BIPIA) | 99.45% |

### Table R4 — Defended ASR (Paper B, InstructDetector)

| LLM | Defended ASR |
|-----|-------------|
| GPT-3.5-Turbo | 0.12% |
| GPT-4o | 0.13% |
| Vicuna-7B | 0.03% |
| Qwen2.5-7B | 0.10% |

---

*Document generated: 2026-06-14*

*Based on:*
- *Paper A: Jingwei Yi et al., "Benchmarking and Defending Against Indirect Prompt Injection Attacks on Large Language Models," KDD 2025 (arXiv:2312.14197)*
- *Paper B: Tongyu Wen et al., "Defending against Indirect Prompt Injection by Instruction Detection," arXiv:2505.06311*
- *Paper C: Yupei Liu et al., "Formalizing and Benchmarking Prompt Injection Attacks and Defenses," USENIX Security 2024 (arXiv:2310.12815)*
