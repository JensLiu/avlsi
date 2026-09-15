# Dossier #9 — Dimensionality Reduction for Boolean Functions

> Full-depth research dossier. Sources: the proposal PDF, the professor's project page (Cortadella, *Dimensionality reduction for Boolean functions*), and related literature on Boolean decomposition and approximate synthesis.

---

## 1. Problem statement

**Input**

- A Boolean function $f: \{0,1\}^n \to \{0,1\}$ (given as a truth table / BDD / DNF), and a target dimension $k < n$.

**Output**

- A subset $S \subseteq \{x_1,\dots,x_n\}$ with $|S| = k$, and a function $g: \{0,1\}^k \to \{0,1\}$ on those variables that **best approximates** $f$.

**Objective**

- Minimize the **approximation error** — the number of input assignments where $g(x_S) \ne f(x)$ (Hamming distance over the truth table).

**The open question (verbatim from the professor):**

> *"Given an n-variable Boolean function, what is the subset of k variables that can be used to find the best approximation of the function?"* (e.g., 20-variable function, 6-variable approximation)

**Conjecture (motivating it):** good variable-subset approximations are a strong *estimator* of decomposition quality — if $f$ is well-approximated by few variables, then $f$ has a good decomposition where those variables form a **bound set** (Ashenhurst–Curtis style). The approximation thus serves as a *fast oracle* for guiding decomposition search.

**Why it's hard.** There are $\binom{n}{k}$ subsets; for $n=20,k=6$ that's ~38k (enumerable), but for realistic $n$ it explodes. There is no known polynomial algorithm; the project is genuinely open (it's a follow-up to Lucas Machado's PhD thesis, with possible publication).

---

## 2. Core idea

Fix a subset $S$. The **best** approximation $g$ on $S$ is easy to find: for each of the $2^k$ assignments of $S$, the $2^{n-k}$ remaining ("free") assignments form a block, and the optimal $g$ outputs the **majority** value of $f$ on that block (ties broken arbitrarily). The error contributed by that block is

$$\text{(block error)} = \min(\#\text{zeros}, \#\text{ones}) \text{ of } f \text{ restricted to the block.}$$

So the whole problem reduces to choosing $S$ that maximizes the **purity** of the partition of the truth table induced by $S$:

$$\arg\min_{|S|=k} \sum_{\alpha \in \{0,1\}^k} \min\big(\#\{x: x_S=\alpha,\, f(x)=0\},\; \#\{x: x_S=\alpha,\, f(x)=1\}\big).$$

**Intuition:** a "good" subset is one whose values almost determine $f$ — i.e., the variables in $S$ carry most of the *information* about the output. This is exactly Shannon cofactoring structure: $f$'s Shannon expansion on the variables in $S$ has cofactors that are *nearly constant*.

---

## 3. Algorithms overview & comparison

Since the problem is open, there's no canonical algorithm — instead a spectrum of *candidate strategies*:

| Approach | Mechanism | Optimal? | Scalability | Notes |
|---|---|---|---|---|
| **Exhaustive search** | evaluate all $\binom{n}{k}$ subsets | exact | tiny n only | baseline + ground truth |
| **Greedy forward/backward selection** | add/remove variable that most improves purity | heuristic | medium | fast, suboptimal |
| **Branch-and-bound** | prune subsets via error lower bounds | exact (with bound) | small–medium | rigorous |
| **Information-theoretic scoring** | rank variables by mutual information / entropy | heuristic | large | fast pre-filter |
| **mRMR-style feature selection** | max-relevance, min-redundancy | heuristic | large | ML-inspired |
| **Spectral / relaxation** | embed & select via eigen/spectral criteria | heuristic | medium | exploratory |

**Key trade-off:** exhaustive is exact but only for small $n$; greedy/scoring scale but may miss *joint* variable interactions (two variables that are only useful together). The interesting research contribution would be (a) characterizing *when* greedy is optimal, and (b) finding a principled heuristic with provable guarantees.

---

## 4. Algorithm detail

### 4.1 Exact evaluation of a subset's error

For a candidate $S$ (size $k$), build a table of $2^k$ counters: iterate all $2^n$ input assignments, accumulate counts of $f=0$ / $f=1$ per block, then sum $\min(\text{count}_0,\text{count}_1)$.

```
subset_error(f, S):
  counts[2^k][2] = 0
  for x in {0,1}^n:
    a = project(x onto S)          # index = bits of x_S
    counts[a][f(x)] += 1
  return Σ_a min(counts[a][0], counts[a][1])
```

- Cost $O(2^n)$ per subset (dominant term). For exhaustive search this is $O(2^n \binom{n}{k})$ — feasible only for small $n$ (or via a smarter incremental/DNF counting).

### 4.2 Exhaustive search

Enumerate all $k$-subsets (combinatorial generation), call `subset_error`, keep the minimum. Establishes the **ground truth** for small $n$, against which every heuristic is validated.

### 4.3 Greedy forward selection

```
forward_select(f, k):
  S = ∅
  while |S| < k:
    v* = argmin_{v ∉ S} subset_error(f, S ∪ {v})   # max purity gain
    S = S ∪ {v*}
  return S
```

- Cheap and effective when variables are *independently* informative; fails on XOR-like functions where no single variable helps alone (the classic counterexample).

### 4.4 Information-theoretic scoring

For each variable $x_i$, estimate **mutual information** $I(x_i ; f)$ (or the impurity drop if $x_i$ alone were selected). Rank variables, take top-$k$. Very fast, but ignores redundancy and interactions.

### 4.5 Branch-and-bound

Maintain a lower bound on achievable error given a partial assignment of variables (e.g., the current error is a lower bound since adding variables can only reduce it). Prune partial subsets whose lower bound already exceeds the best-known solution.

### 4.6 Connecting to decomposition (the research goal)

The **Ashenhurst–Curtis decomposition** of $f$ with **bound set** $B$: $f = h(g_1(x_B), \dots, g_r(x_B), x_{B^c})$. A subset $S$ with small approximation error is a *candidate bound set* — if $f$ is nearly determined by $S$, then the "free" variables contribute little, suggesting a decomposition $f \approx h'(x_S)$ plus a small correction. The project's circuit example shows exactly this: $g$ (red box, 3 variables) + a simple blue box implements $f$.

**The deliverable hypothesis to test:** *does minimizing subset-approximation error correlate with minimizing post-decomposition implementation cost (gates/LUTs)?* If yes, subset-error is a cheap **quality estimator** to steer decomposition.

---

## 5. State of the art

- **Related decomposition work:** Ashenhurst–Curtis and **bi-decomposition** ($f = g \,\mathrm{op}\, h$); modern LUT mappers use fast ACD up to ~16 inputs (IWLS 2023, Mishchenko).
- **Approximate logic synthesis (ALS):** approximating $f$ by a cheaper function (error-tolerant apps); "approximate disjoint bi-decomposition" (Qian et al.) is closely related in spirit.
- **Feature selection:** mRMR and mutual-information ranking from ML are directly applicable as baselines.
- **This project's niche:** none of the above focuses on *variable-subset selection as a decomposition-quality estimator*, which is the open contribution. It's a follow-up to Machado's PhD on FPGA decomposition (Machado 2015).

---

## 6. Key references

- Cortadella, *Dimensionality reduction for Boolean functions* (project page) — the authoritative framing.
- Machado, *PhD thesis on logic decomposition for FPGA mapping* (UPC) — the work this follows.
- Ashenhurst (1959); Curtis (1962) — functional decomposition theory.
- Mishchenko et al., *Boolean decomposition revisited*, IWLS 2023 (fast ACD up to 16 inputs).
- Yao, Huang, Wang, Wu, Qian, *Approximate disjoint bi-decomposition*, 2020s (approximate synthesis).
- Peng, Long, Ding, *Feature selection based on mutual information* (mRMR), IEEE TPAMI 2005 (ML baseline).
- Boole (1854) — expansion theorem; Shannon (1938) — cofactors.

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★★ — the problem is crisp (combinatorial optimization over subsets) but the *solution* is open; the depth is in designing/pruning the search and proving properties.
- **Effort:** high — you must define the error oracle, implement search strategies, run experiments, and (ideally) produce a *finding*.
- **Risk:** **high** — success criteria are fuzzy (research-flavored); you may end with "we characterized greedy vs. exhaustive on X classes." Publication is possible but not guaranteed.
- **Course fit:** good but unconventional — it's a *research* project, not an "implement this known algorithm" project. Requires comfort with exploratory conclusions and Boolean algebra/BDDs.

**Practical project scope (if chosen):** implement exact exhaustive + greedy + information-theoretic selection over truth tables/BDDs (pyeda), benchmark on random + structured functions (adders, parity, symmetric), and empirically test the **decomposition-quality correlation** conjecture using yosys/ABC gate counts. This is the highest-risk but potentially highest-novelty option.
