# Dossier #5 — Cell Placement with Metaheuristics

> Full-depth research dossier. Sources: Kahng et al. *VLSI Physical Design* §4.3.3, Kirkpatrick et al. [KGV83], Boettcher & Percus [BP03].

---

## 1. Problem statement

**Input**

- A **netlist**: cells + multi-pin nets (hyper-edges).
- A **grid** of $n \times m$ slots; each cell occupies exactly **one slot** (the project's simplification).

**Output**

- An assignment of every cell to a distinct slot (a **placement**).

**Objective (minimize)**

- **Total wirelength**, approximated by **HPWL** (half-perimeter wirelength): for each net, $(\max x - \min x) + (\max y - \min y)$ over its pins; sum over nets (optionally weighted).

**Why it's hard.** Even on a grid, minimizing HPWL is a quadratic-assignment-like problem — NP-hard. Metaheuristics are the natural tool: they trade optimality for a good solution in bounded time.

> Note: this project overlaps conceptually with graph drawing (#1) and is the "stochastic" branch of global placement (#4). Here the cell-equals-one-slot assumption makes it a clean *discrete* optimization playground for **Simulated Annealing (SA)** vs. **Extremal Optimization (EO)**.

---

## 2. Core idea

Two complementary local-search philosophies:

- **Simulated annealing (SA):** accept *random* moves, occasionally accepting worse ones with probability $e^{-\Delta/T}$ to escape local minima; cool slowly.
- **Extremal optimization (EO):** always *fix the worst part* — remove the cell with worst "fitness" (largest local cost) and place it elsewhere, letting "avalanches" of re-assignments explore the landscape.

The project is essentially a **controlled experiment**: implement both, compare wirelength, runtime, and robustness.

---

## 3. Algorithms overview & comparison

| Algorithm | Move strategy | Acceptance | Escapes local minima via | Tuning | Notes |
|---|---|---|---|---|---|
| Simulated annealing | random perturb (move/swap) | Metropolis $e^{-\Delta/T}$ | temperature | cooling schedule | classic, proven |
| Extremal optimization | remove *worst-fitness* cell, re-place | always accept new | targeting extremes | τ (rank distribution) | newer, less common |
| τ-EO | same, but pick rank $k$ w.p. $\propto k^{-\tau}$ | always accept | τ controls greediness | τ | generalizes EO |

**Key trade-off:** SA's randomness is *temperature-controlled* (global knob); EO's randomness is *rank-controlled* (local, per-variable fitness). SA is slow but principled; EO is fast and often surprisingly strong, with fewer parameters. A head-to-head comparison is exactly the interesting deliverable.

---

## 4. Algorithm detail

### 4.1 Cost function (HPWL)

For a placement $P$ and nets with (optionally weighted) pins:

$$\Gamma_1 = \sum_{net} w_H(net)\cdot\big(\max_i x_i - \min_i x_i\big) + w_V(net)\cdot\big(\max_i y_i - \min_i y_i\big)$$

HPWL is exact for 2–3 pin nets, ~8% error vs. Steiner for real circuits, and O(1) per net with bounding-box bookkeeping (incremental recomputation makes each move evaluation cheap).

### 4.2 Simulated annealing

(Full treatment in dossier #4, §4.4; summarized here for the grid setting.)

```
SA(V):
  T = T0; P = random placement
  while T > Tmin:
    while not equilibrium:
      newP = PERTURB(P)              # move a cell, or swap two cells
      Δ = COST(newP) - COST(P)       # incremental HPWL
      if Δ < 0 or rand() < exp(-Δ/T): P = newP
    T = α·T                          # α ≈ 0.8–0.95
```

- **Perturb:** MOVE (cell → empty slot, within a shrinking window) or SWAP (two cells).
- **Cost:** HPWL (+ overlap penalty + row-length penalty in TimberWolf's richer version).
- **Cooling:** start hot, slow down mid-way, "quench" at the end; equilibrium via fixed moves/cell or a target acceptance ratio (~44%, Lam).

### 4.3 Extremal optimization

**Idea (Boettcher & Percus).** Assign each cell a local **fitness** = (negative of) its contribution to the objective. A good global solution is one where *all* local contributions are good; conversely, eliminate the *worst* local contributions.

For placement, a natural **fitness** for cell $i$:

$$\lambda_i = - \sum_{net \ni i} \frac{\text{(HPWL contribution of } i \text{ to } net)}{\text{(pins in } net)}$$

(the per-net HPWL attributed to $i$, normalized). A cell with large positive contribution (long nets) is "unfit."

```
EO(P):
  P = random placement
  repeat:
    for each cell i: λ_i = fitness(i)          # local cost contribution
    j = cell with worst fitness                # (basic EO: k=1)
    # τ-EO: rank cells 1..N by fitness; pick rank k with P(k) ∝ k^(-τ)
    move j to a new slot (best empty slot, or random)
    recompute λ for cells whose nets touch j   # "avalanche" if they change
  until stopping criterion
```

- **Basic EO** always removes the single worst cell (k=1) — can be too greedy and get stuck.
- **τ-EO** selects rank $k$ with power-law probability $P(k) \propto k^{-\tau}$:
  - $\tau \to \infty$: only the worst (greedy, localizes activity).
  - $\tau \to 0$: random (no learning).
  - **intermediate τ** (~1.3–2) balances exploitation/exploration — the sweet spot.
- **Avalanche dynamics:** re-placing one cell changes its neighbors' fitness, cascading updates — the self-organized-criticality flavor that makes EO work.

---

## 5. State of the art

- **SA:** established by [KGV83]; dominated VLSI placement for a decade (TimberWolf). Still used for **detailed placement in small windows** and FPGA/constrained problems; global placement has moved to analytic methods (§4).
- **EO:** introduced by Boettcher & Percus [BP03]; successful on graph bipartitioning, TSP, spin glasses; *not* widely adopted in commercial EDA. This makes the SA-vs-EO comparison a genuinely *interesting* (if niche) experimental contribution.
- Both are "chaotic" (small input change → large output change) — a known weakness vs. analytic placers.

---

## 6. Key references

- [KGV83] Kirkpatrick, Gelatt, Vecchi, *Optimization by simulated annealing*, Science 1983.
- [BP03] Boettcher & Percus, *Extremal Optimization: an evolutionary local-search algorithm*, Springer 2003.
- Boettcher & Percus, *Optimization with extremal dynamics*, Complexity 2002 (τ-EO).
- Sechen & Sangiovanni-Vincentelli, *TimberWolf*, DAC 1988 (SA placement).
- Lam, *An efficient simulated annealing schedule*, Yale 1988 (44% acceptance).
- Textbook: Kahng et al., *VLSI Physical Design*, §4.3.3.

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★★ — the algorithms are simple; the depth is in *incremental HPWL evaluation, move generation, cooling/τ tuning, and rigorous experimental comparison*.
- **Effort:** medium. Both SA and EO are small, self-contained, and forgiving to implement.
- **Risk:** low-medium — SA *always* yields a valid placement and improves with time; EO always yields a valid placement; you essentially cannot "fail."
- **Course fit:** strong — maps to §4.3.3 and the "Placement" block; the SA-vs-EO comparison is a clean, demonstrable experimental study.

**Practical project scope (if chosen):** implement SA and EO over an $n\times m$ grid with incremental HPWL; benchmark on random + structured netlists; report wirelength vs. runtime curves, parameter sensitivity (cooling schedule vs. τ), and convergence. Optionally add a **local-search (greedy swap) baseline** to isolate the metaheuristic contribution.
