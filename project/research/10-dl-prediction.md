# Dossier #11 — Deep Learning for Predicting Circuit-Size Complexity

> Full-depth research dossier. Sources: the proposal PDF, the professor's project page (Cortadella, *Deep learning for logic synthesis*), and ML-for-EDA literature.

---

## 1. Problem statement

**Input**

- A Boolean function $f:\{0,1\}^n\to\{0,1\}$, represented either as a **truth table** (bit string of length $2^n$) or as an **SOP matrix** (a "black-and-white picture" of rows=products × columns=variables, cells ∈ {0,1,*}), or as an AIG.

**Output**

- An **estimate of the implementation complexity** of $f$: e.g., the number of 2-input gates, the number of literals in a factored form, or the number of LUTs after technology mapping.

**Method**

- **Supervised learning.** Generate many random functions, synthesize them with a real tool (Yosys/ABC) to obtain ground-truth complexity labels, and train a neural network to predict complexity from the representation.

**Goal / motivation**

- A **fast, cheap complexity estimator** to *guide decomposition* — instead of actually synthesizing every candidate decomposition (expensive), use the model as an oracle to pick good decompositions quickly. Follow-up to Lucas Machado's PhD on FPGA decomposition.

**Why it's hard / novel.** The core *research question* is whether complexity is even predictable from the truth table/SOP matrix (a structural property that's genuinely hard — it connects to circuit lower bounds). This is a novel, exploratory project (flagged as possible Master-thesis basis).

---

## 2. Core idea

**Treat the Boolean function as an image.** An SOP matrix (products × variables, entries 0/1/*) is literally a small bit-map; a **CNN** can ingest it and regress the complexity label. The CNN learns *structural features* (e.g., don't-care patterns, cube overlaps, implied prime implicants) that correlate with implementation cost — without explicit feature engineering.

For truth-table input, the equivalent is a 1-D vector (or a $\sqrt{2^n}\times\sqrt{2^n}$ reshaped image) fed to an MLP/CNN.

---

## 3. Approaches overview & comparison

| Approach | Input | Model | Pros | Cons |
|---|---|---|---|---|
| **MLP / linear regression** | truth-table bits | dense net | simple baseline | poor for large n |
| **CNN on SOP matrix** (professor's suggestion) | SOP matrix as image | CNN | learns spatial structure | needs SOP→matrix normalization |
| **Spectral / Walsh features + regressor** | Rademacher/Walsh spectrum | GBM/MLP | strong theory, interpretable | hand-crafted features |
| **GNN on AIG** (FuncGNN) | AIG graph | graph neural net | captures structure exactly | needs AIG extraction |

**Key trade-off:** CNN-on-matrix is the professor's *stated* approach (no hand features, "is it just a picture?"), while spectral features + gradient boosting are the *strongest baseline* (Walsh spectrum provably relates to circuit complexity — e.g., spectral translation/linearity measures). A GNN on the AIG is the modern, structure-faithful route.

---

## 4. Algorithm detail

### 4.1 Data generation & labeling pipeline

```
generate_dataset(n, N):
  data = []
  for i in 1..N:
    f = random Boolean function on n vars         # e.g., random truth table,
                                                   # or random SOP, or restricted families
    impl = synthesize(f)                          # Yosys/ABC → AIG → map
    label = count_gates(impl)                     # or #literals, or #LUTs
    data.append((representation(f), label))
  return data
```

- **Randomness model matters:** uniform random truth tables are *unlearnable* (almost all functions are maximally complex, so labels are concentrated); the professor likely means *structured* random functions (random SOPs of bounded size, random factored forms, benchmarks). This is a subtle and important design decision.
- **Labels:** synthesize with Yosys (`abc -c "read; strash; ..."`) to get gate counts; or LUT-mapping with ABC `if -K 6` for FPGA LUT counts.

### 4.2 Representation / featurization

- **SOP matrix:** rows = cubes, columns = variables; entries ∈ {0, 1, *} (negative, positive, absent). Pad/normalize rows to a fixed size; feed as a 2-D image to a CNN.
- **Truth table:** $2^n$-bit vector; optionally reshape to $2^{n/2}\times 2^{n/2}$.
- **Spectral (Walsh) features:** compute the Walsh/Rademacher–Walsh spectrum (Hadamard transform of the ±1 truth table); the spectrum's support/energy distribution is a *proven* complexity indicator. Strong, interpretable, and cheap for moderate n.

### 4.3 Model & training

- **CNN:** conv layers (3×3 kernels) over the SOP-matrix image → flatten → dense → scalar regression (MSE/MAE on gate count).
- **Baselines:** (a) trivial predictor (mean), (b) linear/logistic regression, (c) gradient-boosted trees on Walsh features.
- **Evaluation:** train/test split on held-out *functions*; report MAE, $R^2$, and (crucially) whether the model **ranks** functions correctly (for decomposition guidance, *ordering* matters more than absolute count).

### 4.4 Usage as a decomposition oracle

Once trained, the estimator is plugged into a decomposition search: candidate decompositions are scored by predicted complexity instead of by full re-synthesis — the "quick estimation" that is the project's stated goal.

---

## 5. State of the art

- **ML for EDA** is a hot area: learning-based *placement* (DREAMPlace), *routing*, and *synthesis* heuristics.
- **Neural circuit synthesis** (NeurIPS 2022/2024): fully-explainable neural synthesis of combinational circuits from I/O examples; scalable neural circuit *generation*.
- **"Logic synthesis meets ML — trading exactness for generalization"** (arXiv 2020): ML learns to *approximate* incompletely-specified functions, generalizing beyond the care set.
- **FuncGNN (2025):** graph neural network on AIGs capturing functional semantics.
- **This project's niche:** *complexity estimation* (gate/LUT count prediction) as a decomposition oracle — closest to the professor's framing and Machado's PhD, and distinct from *synthesis* itself.

---

## 6. Key references

- Cortadella, *Deep learning for logic synthesis* (project page) — the authoritative framing (CNN on SOP matrix).
- Machado, *PhD thesis on logic decomposition for FPGA mapping* (UPC) — the work this follows.
- Bryerton / "Neural Combinatorial Logic Circuit Synthesis from I/O Examples" (arXiv 2022; NeurIPS 2024 scalable generation).
- "Logic Synthesis Meets Machine Learning: Trading Exactness for Generalization" (arXiv 2020).
- FuncGNN (2025) — GNN on AIGs.
- Brayton & Mishchenko, *ABC: A System for Sequential Synthesis and Verification* (synthesis/labeling tool).
- Yosys (open synthesis framework).

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★ (mostly ML + tool automation; the "algorithmic component" is thinner — see the course's hard requirement).
- **Effort:** medium-high — data-generation + Yosys/ABC automation + model training is a lot of engineering, but not algorithmically deep.
- **Risk:** medium-high — the *research question* ("is complexity predictable from the representation?") may have a disappointing answer (especially for uniform-random functions); success is fuzzy.
- **Course fit:** moderate — it's novel and thesis-flagged, but to satisfy "significant algorithmic component" you should *add* an algorithmic layer: e.g., **Walsh-spectral feature engineering**, or use the estimator *inside a decomposition search* (making it an algorithmic pipeline, not just a regression).

**Practical project scope (if chosen):** build the data pipeline (random structured functions → Yosys/ABC → gate/LUT labels); train a CNN (and a spectral-feature baseline) to predict complexity; rigorously evaluate ranking quality; and — to strengthen the algorithmic claim — plug the estimator into a greedy decomposition search and show it speeds up FPGA mapping. Flag clearly to the professor how you'll satisfy the "algorithmic component."
