# Dossier #4 — Global Placement

> Full-depth research dossier. Sources: Kahng et al. *VLSI Physical Design* §4.3–4.4, Ozdal placement lecture slides, and SOTA literature (ePlace, RePlAce, DREAMPlace).

---

## 1. Problem statement

**Input**

- A **netlist**: a hypergraph of *cells* (standard cells + possibly macro blocks) connected by *nets* (hyper-edges; a net joins 1 driver to ≥1 receivers).
- **Fixed objects**: I/O pads (pins) and fixed macros with predetermined locations.
- A **die/region** of given dimensions, partitioned into **rows** of equal-height standard cells and **legal sites** (aligned to power/ground rails).

**Output**

- An assignment of every movable cell to a legal (site-aligned, non-overlapping) position.

**Objective (minimize, lexicographically)**

1. **Total wirelength** — almost always approximated by **HPWL** (half-perimeter wirelength = horizontal span + vertical span of a net's bounding box).
2. **Routing congestion** — local wire density should stay under capacity.
3. **Signal delay / timing** (timing-driven placement, §8.3).

**Constraints**

- No overlaps; cells on legal sites within rows; density ≤ target per region.

**Two-stage decomposition**

- **Global placement (GP):** continuous coordinates, overlaps *allowed*. Produces a good approximate distribution. This is the algorithmic heart of the project.
- **Legalization + detailed placement (DP):** snap to legal sites, remove overlaps, then local refinement (swaps/slides).

> **Why it's hard:** GP is NP-hard (it generalizes quadratic assignment). Real designs have millions of cells, so only heuristics/continuous relaxations are feasible. The tension is: minimize wirelength (pulls connected cells together → clustering/overlap) vs. satisfy density/spread (pushes cells apart).

---

## 2. Core idea

Treat placement as **continuous optimization of a differentiable cost function**, where:

- **Wirelength** is modeled by a smooth surrogate (quadratic distance, or the weighted-average model), and
- **Density/overlap** is modeled by a penalty term (or enforced by alternating spreading steps).

This splits all modern placers into a loop: *minimize wirelength* ⇄ *spread cells* until convergence, then *legalize*.

Historically there are three paradigms (Fig. 4.6):

| Paradigm | Idea | Example |
|---|---|---|
| **Partitioning-based** | recursively cut the netlist and the region in half, minimizing cut size | min-cut (Capo) |
| **Analytic** | optimize a smooth mathematical cost via numerical methods | quadratic, force-directed, ePlace/RePlAce |
| **Stochastic** | random moves accepted by a cooling criterion | simulated annealing (TimberWolf) |

---

## 3. Algorithms overview & comparison

| Algorithm | Cost model | Overlap handling | Scalability | Quality | Speed |
|---|---|---|---|---|---|
| Min-cut (Breuer) | cut size per cutline | none inherent (partitioning) | good (hierarchical) | medium | fast |
| Quadratic | squared Euclidean distance | needs separate spreading | excellent (flat) | good | very fast |
| Force-directed | spring/ZFT (quadratic) | ZFT + ripple moves | poor (legacy) | medium | medium |
| Simulated annealing | HPWL + overlap + row-length | implicit (penalized) | poor (small designs) | high (if tuned) | very slow |
| Modern quadratic (FastPlace, simPL, Capo) | quadratic + anchors | interleaved spreading | excellent | good | fast |
| Modern nonlinear (APlace, ePlace, RePlAce) | weighted-average + density penalty | smooth density penalty | good (ePlace: flat, FFT) | best | slow |
| DREAMPlace (GPU) | same as ePlace, PyTorch | same | excellent (GPU) | best | very fast (GPU) |

**Key trade-off:** quadratic placers are simpler/faster; nonlinear placers give better quality but need careful numerical tuning. All leading placers today are **analytic**; min-cut and SA survive in niches (routability, small/mixed-size, FPGA).

---

## 4. Algorithm detail

### 4.1 Min-cut placement (Breuer, 1977)

**Idea.** Recursively bisect the layout region and, simultaneously, bisect the netlist so the number of nets crossing the cutline is minimized. Repeat until each region holds few cells, then place optimally by enumeration.

**Cut-cost heuristic.** Cut a hypergraph by the **KL** or **FM** algorithm (§2.4). FM uses **gain** = reduction in cut size from moving a cell, with a **balance criterion** (area ratio). Because each cut is only a local heuristic, the cutlines are minimized one at a time:

$$\min \psi_P(\mathrm{cut}_1) \to \min \psi_P(\mathrm{cut}_2) \to \cdots$$

**Cutline directions.** *Alternating* (V,H,V,H,…) gives squarish regions; *repeating* (all H then all V) creates rows/columns but poorer wirelength.

```
Min-Cut(G, LA, cells_min):
  P = ∅
  regions = { (G, LA) }                  # assign whole netlist to whole area
  while regions ≠ ∅:
    region = pop(regions)
    if |cells(region)| > cells_min:
      (sr1, sr2) = BISECT(region)        # KL/FM partition: sub-netlist + sub-area
      push(regions, sr1); push(regions, sr2)
    else:
      PLACE(region)                      # optimal: enumerate/branch-and-bound
      P = P ∪ region
  return P
```

**Terminal propagation (Dunlop–Kernighan).** Naive min-cut ignores pins outside the current region. Fix: represent external connections as **dummy nodes** projected onto the cutline (using a rectilinear Steiner tree crossing the boundary). A dummy node "close" to the cutline (middle third) is ignored; otherwise it participates in the gain computation. This steers cells toward their external neighbors.

**Pros/cons.** Fast, hierarchical, tunable (e.g., timing-driven). But: randomized/chaotic (tiny input change → big output change), and optimizing one cutline at a time can create congestion elsewhere.

### 4.2 Quadratic (analytic) placement

**Idea.** Approximate wirelength by the **squared Euclidean distance** and minimize it analytically; handle overlap afterwards.

**Formulation.** Decompose each net into 2-pin edges with weights c(i,j). Cost:

$$L(P)=\tfrac12\sum_{i,j} c(i,j)\big[(x_i-x_j)^2+(y_i-y_j)^2\big]$$

x and y **decouple** into two independent convex quadratic programs. Setting the gradient to zero gives a **linear system**:

$$\frac{\partial L_x}{\partial X} = A X - b_x = 0$$

where:
- $A[i][j] = -c(i,j)$ for $i\neq j$; $A[i][i] = \sum_j c(i,j)$ (the **weighted graph Laplacian**);
- $b_x[i]$ = sum of x-coordinates of *fixed* cells/pads connected to $i$.

$A$ is symmetric positive semi-definite (PSD); solve iteratively by **Conjugate Gradient (CG)** or **Successive Over-Relaxation (SOR)**. Convexity ⇒ any local min is global.

```
QuadraticPlace(V):
  build Laplacian A and rhs b from netlist + fixed pins
  X = CG_solve(A, b_x);  Y = CG_solve(A, b_y)   # seed placement
  repeat:
    X,Y = CG_solve(A + spreading, b)            # solve with spreading forces
    (X,Y) = SPREAD(X,Y)                          # or geometric scaling / anchors
  until converged
  return legalize(X,Y)
```

**Net models.** Multi-pin nets → edges:
- **Clique**: every pin pair, weight $2/p$ (or normalized). No new variables; good for small nets.
- **Star**: all pins connect to a movable/fixed star point at the centroid. Linear #edges; good for large nets.
- **Linearization trick**: reweight each term by $1/|x_i-x_j|$ to turn quadratic → linear, updated between rounds.

**Spreading.** Pure quadratic solution collapses all cells to a point. Two fixes:
1. **Geometric scaling / sorting** — sort cells in dense regions and re-place in order (FastPlace).
2. **Anchor points / fake nets** — add imaginary fixed pins pulling cells out of dense bins; re-solve (FastPlace interleaves scaling then anchored QP).

### 4.3 Force-directed placement

**Idea.** Mechanical analogy: cells = masses, nets = Hooke-law springs. Attraction force ∝ distance. Equilibrium position of cell $i$ is its **Zero-Force Target (ZFT)**:

$$F_{ab} = c(a,b)\cdot(\vec b - \vec a),\qquad \vec F_i = \sum_j \vec F_{ij}$$

Setting forces to zero and solving for $x_i^0$:

$$x_i^0 = \frac{\sum_j c(i,j)\,x_j}{\sum_j c(i,j)},\qquad y_i^0 = \frac{\sum_j c(i,j)\,y_j}{\sum_j c(i,j)}$$

i.e., the **weighted centroid of i's neighbors**. Force-directed is a special case of quadratic placement (same objective, solved by fixed-point iteration instead of CG).

```
ForceDirected(V):
  P = PLACE(V)                       # arbitrary initial
  status[*] = UNMOVED
  while not all moved / not stop:
    c = unmoved cell with max degree
    z = ZFT(c)                        # weighted centroid of neighbors
    if z vacant: move c → z
    else: RELOCATE(c)                 # swap / chain move / ripple move
    status[c] = MOVED
```

**Occupied-ZFT resolution** (4 options): move near; **swap** if total HPWL decreases; **chain move** (displace occupant to next slot); **ripple move** (recompute occupant's ZFT, cascading). Ripple-move variant sorts cells by descending degree, locks cells once moved to avoid cycles, and caps "abort" (suboptimal) moves.

**Pros/cons.** Conceptually simple; does not scale; poor at spreading dense regions (modern force-directed = quadratic + repulsive forces, §4.5).

### 4.4 Simulated annealing (TimberWolf)

**Idea.** Stochastic hill-climbing with a temperature-controlled acceptance of worse moves (Metropolis criterion), guaranteeing convergence to a global optimum in the limit.

```
SA(V):
  T = T0; P = PLACE(V)
  while T > Tmin:
    while not equilibrium(T):
      newP = PERTURB(P)
      Δ = COST(newP) - COST(P)
      if Δ < 0: P = newP
      else if rand() < exp(-Δ/T): P = newP
    T = α·T
```

**PERTURB.** MOVE (shift a cell), SWAP (exchange two cells), or MIRROR. Moves restricted to a window $w_T\times h_T$ that shrinks with temperature: $w_{next}=w_{curr}\cdot\frac{\log T_{next}}{\log T_{curr}}$.

**COST (TimberWolf v3.2).** $\Gamma = \Gamma_1 + \Gamma_2 + \Gamma_3$:
- $\Gamma_1$ = weighted HPWL (horizontal/vertical weights $w_H,w_V$);
- $\Gamma_2$ = $\sum o(i,j)^2$ (squared cell **overlap area**, penalizing large overlaps quadratically);
- $\Gamma_3$ = $\sum |L(row)-L_{opt}(row)|$ (row-length deviation from target).

**Cooling.** Start very hot ($T_0 \sim 4\cdot10^6$), α ≈ 0.8 fast phase, α ≈ 0.95 fine-tuning, final "quench" α ≈ 0.8; stop at $T_{min}=1$. Equilibrium via fixed iterations/cell (≈100/cell for ~200 cells) or a target acceptance ratio (Lam's 44%).

**Pros/cons.** Can find global optimum given time; robust to arbitrary constraints (FPGA, mixed-size); but very slow, needs laborious tuning, chaotic. Today used for **detailed placement in small windows** and constrained/FPGA problems — not global placement.

### 4.5 Modern placers (the two analytic families)

**A. Quadratic placers** (FastPlace 3.0, simPL, Capo, mPL6). Solve large sparse Laplacian systems with **CG**, spread by **anchor points**, and interleave spreading with re-solving. Simpler, faster, easier to parallelize; quality catching up (5–6× faster than nonlinear as of 2010).

**B. Non-convex / nonlinear optimizers** (APlace, ePlace, RePlAce). Use a **differentiable wirelength surrogate** — the **weighted-average (WA)** approximation to HPWL (verified in the ePlace paper):

$$f_{W_e}^x(v)=\frac{\sum_{i\in e} x_i e^{x_i/\gamma}}{\sum_{i\in e} e^{x_i/\gamma}}-\frac{\sum_{i\in e} x_i e^{-x_i/\gamma}}{\sum_{i\in e} e^{-x_i/\gamma}}$$

plus a **smooth density penalty**. Optimized by **Nesterov's accelerated gradient** (ePlace/RePlAce; earlier APlace used nonlinear CG). Higher quality, but numerically finicky; ePlace's flat FFT-based density removed the need for clustering.

### 4.6 Legalization & detailed placement (brief)

- **Tetris** (Hill): sort by x, greedily snap each cell to nearest legal site; fast but ignores netlist and can drift.
- **Optimal small-bin**: exhaustive/branch-and-bound on 4–6 cell bins from min-cut (up to ~11 cells).
- **Interleaving**: split a window into two halves, optimally merge preserving intra-half order (up to ~20 cells).
- **FastPlace-DP / ECO-System**: incremental swaps/slides; ECO-System re-runs Capo on overlap-hot regions.

---

## 5. State of the art

The modern lineage (all **nonlinear analytic**, descended from force-directed):

- **ePlace (DAC 2014; journal TODAES 2015)** — *verified primary.* One-stage, flat, nonlinear analytic placer. Innovations: (1) **electrostatic density** — every object is a charge, density = total electric potential energy, solved via **Poisson's equation by FFT** in $O(n\log n)$ (Neumann boundary, zero-frequency removed) ⇒ smooth global density gradient, **no netlist clustering**; (2) **Nesterov's method** with **Lipschitz-constant-predicted steplength** (inverse Lipschitz + backtracking); (3) diagonal **preconditioner** $|E_i|+\lambda q_i$ equalizing macros vs. standard cells; (4) an **SA-based macro legalizer (mLG)** + second-phase standard-cell GP (cGP). Cost $f = f_W + \lambda N$; wirelength by the **weighted-average** model. Results: 2.83%/4.59%/7.13% shorter wirelength than BonnPlace/MAPLE/NTUplace3-unified on ISPD'05/ISPD'06/MMS, and smallest density overflow.
- **RePlAce (TCAD 2018)** — *verified primary.* ePlace-style Nesterov engine + two levers: (1) **constraint-oriented local density** (per-bin over-demand smoothing); (2) **dynamic step-size adaptation** via HPWL "transition points" found by a *trial placement* (tGP). Adds **routability-driven placement**: invoke global router (NCTU-GR) for congestion, then **cell inflation** $(\text{demand}+\text{blk})/\text{cap}$ raised to $\gamma_{super}=2.33$ (capped 2.5×). Results: 2% (ISPD'05/06) and 2.73% (MMS) HPWL reduction over best-known; 8.5–9.59% sHPWL reduction on DAC/ICCAD'12 routability suites. C++, single engine, no benchmark tuning.
- **DREAMPlace (DAC 2019)** — reframes ePlace's placement as **training a neural network**; implemented in **PyTorch** with custom CUDA wirelength/density kernels; ~**30× speedup** over CPU RePlAce on ISPD'05. Runs on CPU/GPU.
- **DREAMPlace 3.0 (ICCAD 2020)** — multi-electrostatics + region constraints for robustness/versatility.
- **DG-RePlAce (2024)** — GPU-accelerated, built on OpenROAD, exploits datapath/dataflow structure of ML accelerators.
- **AutoDMP (2023) / dynamic algorithm configuration** — automatically tune placer hyperparameters (DREAMPlace-based) to fix the known "manual tuning is painful" problem.

**Open issues:** ePlace-family placers can be fragile across workloads; tuning is hard; ML-accelerator-like designs (2D PE arrays) stress scalability and QoR.

**Free tools:** APlace, Capo, FastPlace 3.0, mPL6, simPL, RePlAce, DREAMPlace; OpenROAD integrates RePlAce/DREAMPlace for full-flow.

---

## 6. Key references

- [Bre77] Breuer, *A class of min-cut placement algorithms*, DAC 1977.
- [KGJA91] Kleinhans, Sigl, Johannes, Antreich, *GORDIAN: VLSI placement by quadratic programming and slicing optimization*, IEEE TCAD 1991.
- [KGV83] Kirkpatrick, Gelatt, Vecchi, *Optimization by simulated annealing*, Science 1983.
- Sechen & Sangiovanni-Vincentelli, *TimberWolf*, DAC 1988.
- Dunlop & Kernighan, *Terminal propagation* (external pins in min-cut).
- Viswanathan & Chu, *FastPlace 3.0*, ICCAD 2004 (quadratic + anchors).
- Kim, Lee, Markov, *simPL*, ICCAD 2010.
- Chan et al., *mPL6*; Kahng & Wang, *APlace*.
- Lu et al., *ePlace: Electrostatics-based placement*, DAC 2014.
- Cheng et al., *RePlAce*, IEEE TCAD 2018.
- Lin et al., *DREAMPlace*, DAC 2019; *DREAMPlace 3.0*, ICCAD 2020.
- Textbook: Kahng/Lienig/Markov/Hu, *VLSI Physical Design*, §4.

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★★★ — convex optimization, numerical linear algebra (CG), graph partitioning (FM), density/spreading, plus legalization.
- **Effort:** high. A faithful **min-cut** placer (FM + terminal propagation) is tractable; a **quadratic** placer (Laplacian + CG + anchors) is a solid mid-size project; full **ePlace/DREAMPlace** (electrostatics + Nesterov + GPU) is graduate-level and not expected.
- **Risk:** min-cut medium; quadratic medium (CG + spreading are well-specified); nonlinear high (numerical tuning, legalization).
- **Course fit:** perfect — maps to §4 of the textbook and the "Placement" syllabus block; strong presentation potential (visual placements, HPWL curves, runtime scaling).

**Practical project scope (if chosen):** implement *either* min-cut (FM-based) *or* quadratic (Laplacian + CG + anchor-point spreading) + Tetris legalizer, evaluate HPWL vs. benchmark netlists. Optional extension: compare the two, or add congestion/timing weights.
