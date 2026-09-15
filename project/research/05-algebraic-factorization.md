# Dossier #10 — Algebraic Factorization

> Full-depth research dossier. Sources: De Micheli *Multi-Level Logic Synthesis* lecture slides (DT11), Brayton [Bra87], Golumbic & Mintz [GM99].

---

## 1. Problem statement

**Input**

- A Boolean function given as a **sum of products (SOP)**, e.g.:
$$F = adf + aef + bdf + bef + cdf + cef + bfg + h$$

**Output**

- A **factored (multi-level) form** — a parenthesized algebraic expression with fewer literals, e.g.:
$$F = (a+b+c)(d+e)\,f + bfg + h$$

**Objective (minimize)**

- **Literal count** (primary area estimator) and/or **number of levels / stages** (delay estimator). Optionally power.

**Constraint / model**

- **Algebraic** methods treat expressions as polynomials (no Boolean identities like $x + \bar x = 1$ or $x\cdot x = x$); **Boolean** methods may also exploit these identities and don't-care conditions.

**Why it's hard.** Even "simple" multi-input single-output network optimization is computationally hard; few exact methods exist. The goal is to find *common sub-expressions (divisors)* across a network and extract them — the core sub-problem (finding the best kernel/divisor) reduces to **rectangle covering** in a Boolean matrix, which is itself NP-hard in general.

---

## 2. Core idea

**Division drives factorization.** If a function $f_{dividend} = f_{divisor}\cdot f_{quotient} + f_{remainder}$ with disjoint variable supports, then the divisor is a reusable sub-expression. The central question is: *which divisors exist, and which are good?*

**Kernel theory (Brayton–McMullen)** answers it: the useful multiple-cube divisors of a function are exactly its **kernels** (cube-free quotients by **co-kernels**), and *two functions share a common divisor iff their kernel sets intersect*. So factorization becomes:

1. Compute kernels (recursively).
2. Represent kernels as a **cube×variable incidence matrix**; a **prime rectangle** (all-1s submatrix) = a common divisor.
3. **Extract** the best rectangle (largest literal savings), introduce a new node, iterate.

---

## 3. Algorithms overview & comparison

| Approach | Mechanism | Optimal? | Cost | Notes |
|---|---|---|---|---|
| **Algebraic division** | literal-matching quotient/remainder | exact for a given divisor | O(n log n) | fast, core primitive |
| **Kernel extraction (Brayton)** | recursive kernels + rectangle covering | heuristic (greedy) | medium | the classic, widely used |
| **Boolean factoring** | kernels + rectangle covering with Boolean relations | better QoR | high | ISCAS 1990, harder |
| **Graph partitioning (GM99)** | model factoring as graph partition | heuristic | medium | alternative to rectangle covering |
| **Double-cube / single-cube extraction** | restricted small kernels | heuristic | low | efficient, testability-preserving |
| **ABC `fx` command** | modern fast algebraic factoring | heuristic | low | industrial-strength |

**Key trade-off:** algebraic methods are fast/robust but miss Boolean opportunities (e.g., $a+\bar a b \to a+b$); Boolean methods get better results but are expensive and irreversible. In practice the pipeline is: **algebraic extraction for speed, Boolean techniques (don't cares) for the last few %**.

---

## 4. Algorithm detail

### 4.1 Algebraic division

Divide SOP $A$ by SOP $B$ (both as sets of cubes). Algebraic quotient exists only if supports are disjoint.

```
ALGEBRAIC_DIVISION(A, B):        # A = dividend cubes, B = divisor cubes (n cubes)
  for i = 1..n:
    D = { C_A[j] : C_A[j] ⊇ C_B[i] }        # cubes containing divisor cube i
    if D == ∅: return (∅, A)                 # no division possible
    D_i = D with variables of sup(C_B[i]) dropped
    if i == 1: Q = D_i
    else:      Q = Q ∩ D_i
  R = A − Q×B
  return (Q, R)
```

**Filter theorem** (cheap pre-check before dividing): $f_i/f_j$ is empty if $f_j$ contains a variable not in $f_i$, or a cube whose support isn't contained in any cube of $f_i$, or more terms, or a higher variable count than $f_i$.

### 4.2 Kernels and co-kernels

- **Cube-free:** an expression not factorable by a single cube (e.g., $a+bc$ is cube-free; $ab+ac = a(b+c)$ is not).
- **Kernel:** a cube-free quotient $f/C$ where $C$ is a **co-kernel** (a cube). $K(f)$ = set of all kernels.
- Example: $f = ace + bce + de + g$: divide by $ce \to a+b$ (kernel); divide by $e \to ac+bc+d$ (kernel); $K(f)=\{(a+b),(ac+bc+d), ace+bce+de+g\}$.

**Brayton–McMullen theorem:** $f_a$ and $f_b$ share a multiple-cube divisor iff some $k_a\in K(f_a)$, $k_b\in K(f_b)$ have a non-empty intersection. ⇒ if kernel sets don't intersect, no common sub-expression exists.

### 4.3 Recursive kernel computation

**Idea:** kernels of kernels are kernels, and multiplication is commutative (so deduplicate by a pointer $j$ over literals).

```
KERNELS(f, j):
  K = ∅
  for i = j..n:                       # literals x_i in support, in order
    if |CUBES(f, x_i)| ≥ 2:           # x_i appears in ≥2 cubes
      C = maximal cube containing x_i such that CUBES(f,C)==CUBES(f,x_i)
      if C has no variable x_k with k < i:      # avoid duplicates
        K = K ∪ KERNELS(f/C, i+1)
  K = K ∪ f
  return K
```

where `CUBES(f,C)` returns the cubes of $f$ containing cube $C$. Complexity is polynomial in practice (Brayton); this recursive form still outperforms alternatives.

### 4.4 Rectangle covering (matrix representation)

Build the **incidence matrix** (cubes × variables, entries ∈ {0,1}). A **rectangle** = a submatrix of all-1s; a **prime rectangle** = maximal such rectangle.

- A **co-kernel is a prime rectangle with ≥2 rows** (e.g., rows {1,2}, columns {3,5} = co-kernel $ce$ for $f=ace+bce+\dots$).
- **Single-cube extraction:** tag each expression with a fresh variable, form the auxiliary sum, find the largest co-kernel present in ≥2 tagged expressions, extract it.
- **Multiple-cube (kernel) extraction:** relabel cubes as new variables; kernels become cubes in these new variables; a prime rectangle in the relabeled kernel matrix = a common kernel → extract it.

```
EXTRACT(expressions):
  build aux function = union of all expressions (tagged)
  repeat:
    find best prime rectangle (co-kernel) appearing in ≥2 expressions
    if none beneficial: stop
    create new node z = extracted divisor
    substitute z back into all expressions
```

**Finding the best rectangle (Bra87 heuristics):** **GREEDR** grows a rectangle from a seed row, **GREEDC** from a seed column, and **PING-PONG** alternates row/column growth to find a low-weight prime rectangle; weight = (co-kernel + kernel literal cost) / (#care terms covered).

### 4.5 Kernel-based decomposition & extraction

- **Decomposition:** pick a kernel (e.g., $ac+bc+d$ of $f=ace+bce+de+g$), decompose $f = te+g$ with $t=ac+bc+d$, then recur on $t$: $t = sc+d,\ s=a+b$ ⇒ $f=(sc+d)e+g$.
- **Double-cube extraction:** restrict to 2-cube kernels / 2-literal single-cube kernels (and complements) — cheap, testability-preserving, very effective.
- The project example $F=(a+b+c)(d+e)f+bfg+h$ is precisely a kernel extraction (co-kernel $f$, kernel $(a+b+c)(d+e)$, then the remaining $bfg$ term is left as is).

### 4.6 Graph-partitioning alternative [GM99]

Model factoring as a **graph problem**: cubes are vertices, weighted by shared literals; a factored form corresponds to a recursive bipartition/biclique structure of this graph, and graph-partitioning heuristics find good divisors. Conceptually parallel to rectangle covering, but the search is driven by partition cost rather than prime-rectangle enumeration.

---

## 5. State of the art

- **Brayton's kernels/co-kernels + rectangle covering** [Bra87] (*verified primary*) remains the canonical method. Bra87 defines a **speed/quality spectrum of factorers**: **LF** (literal, fastest) → **QF** (quick factor, one level-0 kernel) → **XF** (all kernels) → **BF** (Boolean division) → **DF** (DeMorgan/duality), plus **OF** (optimal factoring via rectangle covering with **PING-PONG / GREEDR / GREEDC** heuristics). **QF** became the workhorse for fast area/delay estimation.
- **Boolean factoring** (ISCAS 1990) extends it with Boolean relations for better quality at higher cost.
- **Industrial reality:** the Berkeley **ABC** tool's `fx` command implements fast algebraic factoring/decomposition (based on this theory) and is a standard front-end for AIG-based synthesis; **SIS/MVSIS** are the academic descendants of this line.
- Modern synthesis is increasingly **AIG-based** (structural hashing + cut rewriting, e.g., ABC `rewrite`/`refactor`), with algebraic factoring used as a *pre- or post-pass* rather than the sole engine.

---

## 6. Key references

- [Bra87] Brayton, *Factoring logic functions*, IBM J. Research & Development 31(2), 1987 (the project's cited paper).
- Brayton & McMullen, *The decomposition and factorization of Boolean expressions*, ISCAS 1982 (kernel theory).
- Brayton, Hachtel, Sangiovanni-Vincentelli, *Multilevel logic synthesis* (MIS/SIS), Proc. IEEE 1990.
- [GM99] Golumbic & Mintz, *Factoring logic functions using graph partitioning*, ICCAD 1999 (the project's alternative).
- De Micheli, *Synthesis and Optimization of Digital Circuits*, McGraw-Hill 1994 (textbook).
- Mishchenko, Chatterjee, Brayton, *DAG-aware AIG rewriting* (ABC), DAC 2006.

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★★★ — algebraic division, recursive kernel theory, rectangle covering (NP-hard), and a real optimization loop.
- **Effort:** high. A working algebraic-division + single-cube extraction is moderate; full **kernel extraction + rectangle covering** is a substantial but classic build.
- **Risk:** high — many sub-steps (kernel enumeration, rectangle finding, substitution bookkeeping) with subtle algebra; no crisp oracle to verify optimality. Best scoped as "good factorization," not "optimal."
- **Course fit:** strong — maps to the "Multi-level logic synthesis" syllabus block (De Micheli slides); a well-chosen example (like the project's own $F$) makes a clean demo.

**Practical project scope (if chosen):** implement algebraic division + recursive kernel computation + a rectangle-covering extractor; evaluate literal savings vs. input SOPs; optionally compare single-cube vs. multiple-cube vs. double-cube extraction, or add a graph-partitioning variant [GM99].
