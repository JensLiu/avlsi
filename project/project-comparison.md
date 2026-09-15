# AVLSI — Project Selection Comparison

> Source: `projct-proposals.pdf` (Jordi Cortadella, Sept 2025) + `Guia_Docent.pdf`.

## What's at stake

From the course guide: **Grade = 35% FP (project) + 35% FT (final test) + 30% EX (2 exercises)**.

- The project is the single biggest graded component.
- Requires a **public in-class presentation**.
- Professor explicitly says to **start early**.
- Budget: ~20h guided self-study minimum, realistically more.
- Hard requirement repeated across all proposals: **significant algorithmic component** — this is an algorithms course, not "script an EDA tool".

## What's actually expected: implement vs. innovate

**Short answer: mostly implement (well), with experimentation and understanding — not invent something new.** But it's not a flat "just code it" either.

### What the documents literally say

> "The student will have to implement an algorithm to solve a specific Electronic Design Automation (EDA) problem." *(course guide, "EDA project" activity)*

> "It is essential that the projects contain a **significant algorithmic component**." *(proposal PDF)*

> "The students are also **encouraged to propose other projects**… not necessarily in the area of EDA, but somehow related to the challenges faced in EDA problems." *(proposal PDF)*

The contract is **implementation**, not original research. The word "innovation" never appears.

### Three tiers of expected depth

**Tier 1 — "Implement this known algorithm faithfully" (most projects):** #1, #2, #3, #5, #8, #10. The PDF hands you a reference paper and a strategy (e.g. "as described in [WL86]" #2, "as described in [MFNK96]" #3, "based on Quine-McCluskey [McC56]" #8). Reimplement correctly, understand deeply, evaluate experimentally. Originality not required.

**Tier 2 — "Implement + choose the approach/model" (some freedom):** #4 ("explore one of these two strategies"), #6 ("an algorithm or a mathematical model must be proposed"), #7 ("an algorithm (or a SAT/ILP model)"). Freedom in *how* to solve, but the problem is fixed and standard.

**Tier 3 — "Genuinely open / invites original work" (only 2):** #9 ("The main question we want to answer is…") is an actual research question. #11 ("a novel project") is flagged as master-thesis material.

### Mapping to grading

Grade is FP 0–10 + public presentation + runnable code reproducing results. Course competences emphasize *modeling, experimental design, analysis and interpretation, reasoning* — none say "novelty".

Realistic bar for a strong grade:
1. Correct, working implementation (not just calling a library).
2. Experimental evaluation — benchmarks, metrics, comparisons, plots.
3. Clear reasoning about why it works / where it breaks.
4. Clean presentation.

Innovation/novelty is **bonus**, not baseline — except for #9/#11 where it's the point.

### Practical takeaway

- Tier-1/2 projects: you are **not** expected to beat state of the art or publish. Faithful implementation + solid experimental study is a perfectly good project.
- Cheapest "innovation" without risk: **add a comparison** the proposal didn't ask for (e.g. #5 SA *vs* EO, or two strategies in #4/#7).
- Only choose #9/#11 if you actually want open-ended research and accept fuzzier success criteria.

---

## Decision criteria

1. **Language comfort** — Almost everything can be done in Python (NetworkX, PyEDA, z3, or-tools). Only analytical placement (#4) and sequence pairs (#3) on large instances really reward C++.
2. **Algorithmic "wow" vs. risk** — #8 is well-specified (exact, low risk). #4, #7, #9, #11 are more open-ended.
3. **Background leverage** — RTL/DSA background leans toward logic synthesis (#8, #9, #10) and placement (#5). Physical routing (#6, #7) is pure graph algorithm work.
4. **Determinism of deliverable** — #2, #3, #6, #7 have *visual* outputs (layouts/routes) that make the presentation concrete. #9 and #11 have fuzzier success criteria.
5. **Benchmarks availability** — #1 has a ready benchmark library; the rest need you to generate inputs (itself part of the work).
6. **Master-thesis upside** — only #11 (and arguably #9) is flagged as thesis material.

---

## The 11 projects in detail

### 1. Graph drawing
- **What you build:** Read a node/edge file, output an aesthetically good 2D layout.
- **The algorithmic heart:** (a) *spectral* — Laplacian eigenvectors as coordinates; (b) *multilevel force-directed* [Hu06] — coarsen, lay out coarse, refine with repulsion/attraction; (c) *Kamada-Kawai* [KK89] — spring model, Newton iteration on a nonlinear system.
- **Depth:** ★★★ (linear algebra / iterative optimization, well-trodden). **Effort:** medium. **Risk:** low. **Language:** Python (numpy for spectral).
- **Impact:** ★★★ — visually rewarding, ready benchmark library, but the least EDA-native (generic CS). Must *implement* the algorithm, not call `networkx.spring_layout`.
- **Verdict:** Solid, safe, but not the most "wow" for a VLSI course. *Research update:* full detail (spectral Laplacian/Fiedler, Fruchterman–Reingold force-directed + multilevel, Kamada–Kawai/stress majorization) in `research/11-graph-drawing.md`. Least EDA-native; self-contained but weakly tied to VLSI concepts.

### 2. Floorplanning with slicing structures
- **What you build:** Pack rectangular blocks into a bounding box minimizing area + wirelength. Output must be **sliceable** (binary tree / normalized Polish expression).
- **The algorithmic heart:** Wong–Liu [WL86] shape functions — bottom-up merge of piecewise-linear shape curves, top-down coordinate assignment — plus search (typically simulated annealing over Polish expressions).
- **Depth:** ★★★★ (recursion + geometric curve arithmetic + metaheuristic search). **Effort:** medium-high (shape functions are fiddly). **Risk:** medium. **Language:** Python.
- **Impact:** ★★★★ — maps 1:1 onto the course "Floorplanning" content; produces a real layout image.
- **Verdict:** Strong, faithful-to-course pick. *Research update:* full detail (slicing tree + Polish expression, shape functions + H/V composition, Stockmeyer sizing, Wong–Liu SA moves) in `research/09-slicing-floorplanning.md`. Key insight: area min is *polynomial* for a fixed slicing tree (shape-function calculus); the search over trees is the NP-hard part.

### 3. Floorplanning with sequence pairs
- **What you build:** Same placement goal, but using the **sequence pair** representation [MFNK96].
- **The algorithmic heart:** Convert a sequence pair into horizontal & vertical constraint graphs, compute x/y via **longest-path / topological sort**, then pack. Search over sequence pairs with simulated annealing.
- **Depth:** ★★★★½ (constraint-graph construction is genuinely subtle). **Effort:** medium-high. **Risk:** medium. **Language:** C++ helps on large instances.
- **Impact:** ★★★★★ — sequence pairs are what modern placers actually use; strongest algorithmic story in the physical-design block.
- **Verdict:** Top-tier. Harder than #2, more impressive. *Research update:* full detail (sequence-pair rules, constraint graphs, weighted-LCS decoding, SA move set, comparison with B*-tree/O-tree/TCG) in `research/04-sequence-pair-floorplanning.md`. Key insight: decoding = weighted LCS in O(n²) (or O(n log n) fast), and every sequence pair is a legal floorplan, so SA never repairs overlaps.

### 4. Global placement
- **What you build:** Distribute netlist cells uniformly across the die minimizing wirelength. Two forks:
  - **Min-cut** [Bre77]: recursive bipartitioning (Fiduccia-Mattheyses / Kernighan-Lin) to slice the die.
  - **Analytical** (GORDIAN [KGJA91]): quadratic programming formulation, sparse linear solve for x/y, recursive partition to legalize/spread.
- **Depth:** min-cut ★★★★; analytical ★★★★★ (sparse solvers, conjugate gradient, legalization). **Effort:** high. **Risk:** min-cut medium; analytical **high** (legalization is where students stall). **Language:** C++ or Python+numpy.
- **Impact:** ★★★★★ — placement is the center of physical design; GORDIAN is a landmark.
- **Verdict:** Min-cut fork = realistic and strong; analytical = ambitious. *Research update:* full detail (FM min-cut, quadratic/CG, force-directed, SA, ePlace/DREAMPlace SOTA) in `research/01-global-placement.md`. The "analytical" fork is best scoped as **quadratic placement** (Laplacian + CG + anchor spreading + Tetris legalizer) — a tractable mid-size project — rather than full nonlinear ePlace.

### 5. Cell placement (metaheuristics)
- **What you build:** Place cells on an n×m grid (one cell per slot), minimize half-perimeter wirelength (HPWL).
- **The algorithmic heart:** Simulated Annealing [KGV83] (move generation, HPWL cost, cooling schedule) and/or Extremal Optimization [BP03] (fitness ranking, τ-selection) — plus a clean **comparison study** between them.
- **Depth:** ★★★★ (metaheuristic design + experimental rigor). **Effort:** medium. **Risk:** low-medium. **Language:** Python.
- **Impact:** ★★★★ — self-contained, natural "SA vs EO" experimental story.
- **Verdict:** Best *balanced* pick. Interesting and hard to fail at. *Research update:* full detail (HPWL cost, SA with move/swap + cooling, EO with fitness + τ-EO power-law rank, avalanche dynamics) in `research/07-cell-placement.md`. Key insight: SA and EO are both easy, valid-placement guarantees; the depth is in incremental HPWL + a rigorous SA-vs-EO comparison.

### 6. Global routing
- **What you build:** On a grid, connect same-colored terminals (each color = one net) with wires, subject to **per-edge capacity**; minimize total wirelength.
- **The algorithmic heart:** Per-net Steiner-tree approximation + overflow minimization, as heuristic (rip-up & reroute) or mathematical model (ILP / min-cost flow).
- **Depth:** ★★★★½ (Steiner trees + capacity/overflow). **Effort:** high. **Risk:** medium-high (legal problem is NP-hard; deliver heuristics + measure overflow). **Language:** Python + or-tools/z3.
- **Impact:** ★★★★★ — classic EDA, visual grid output, strong "algorithm + model" narrative.
- **Verdict:** Excellent if you enjoy graph optimization and want a real challenge. *Research update:* full detail (RSMT/Hanan, Dijkstra/A*, ILP route-selection, rip-up & reroute, negotiated congestion, FastRoute/NTHU-Route) in `research/03-global-routing.md`. Key insight: the "algorithm or mathematical model" choice maps to **NCR heuristic vs. ILP** — a natural comparison scope.

### 7. Detailed routing
- **What you build:** On a 2D grid, connect node-pairs along grid edges with **no shorts**. Simplified from [CPGM14]; "Flow Free"-like.
- **The algorithmic heart:** (a) **maze routing** (Lee/BFS) per net with rip-up-and-reroute, or (b) encode legality as **SAT/ILP** and solve with z3/or-tools (provably correct routes).
- **Depth:** ★★★★ (pathfinding + ordering, or clean CNF encoding). **Effort:** medium. **Risk:** medium (SAT/ILP route is tractable and elegant). **Language:** Python + z3.
- **Impact:** ★★★★★ — visually satisfying, unambiguous correctness test, strong modeling angle.
- **Verdict:** Best *interest-to-effort* ratio. Strongly recommend the SAT/ILP variant. *Research update:* full detail (Lee/BFS, Dijkstra, A*, HCG/VCG + left-edge/dogleg, SAT/ILP encoding, TritonRoute/Dr. CU) in `research/02-detailed-routing.md`. Key insight: the SAT/ILP (multi-commodity-flow) variant gives *provable* legality and is the highest-return scope; A* + rip-up-reroute is the more algorithmic alternative.

### 8. Two-level minimization (Quine-McCluskey)
- **What you build:** Exact SOP minimizer: generate all prime implicants, then select a minimum cover.
- **The algorithmic heart:** QM combination by group of #-of-ones, then exact cover via **Petrick's method or branch-and-bound** (set cover is NP-hard — the cover solver is the interesting part), with don't-care handling.
- **Depth:** ★★★½ (exact algorithms, branch-and-bound). **Effort:** low-medium. **Risk:** low. **Language:** Python (PyEDA for verification).
- **Impact:** ★★★ — foundational and exact, ties to course content (Espresso is its heuristic cousin), but less visually impressive.
- **Verdict:** Safest clean deliverable with real algorithmic content. *Research update:* full detail (QM prime generation, prime table reductions, Petrick, branch-and-bound EXACT_COVER, Espresso EXPAND/REDUCE/IRREDUNDANT) in `research/08-two-level-minimization.md`. Key insight: the interesting part is the NP-hard set-cover phase, not the prime generation.

### 9. Dimensionality reduction for Boolean functions
- **What you build:** Given n-variable f, find the **best k-variable subset** to approximate f (fewest mismatches) — and *explain why* (cofactor/entropy structure).
- **The algorithmic heart:** Exact search over C(n,k) subsets for small n + heuristic/pruning for larger n; connect to decomposition quality.
- **Depth:** ★★★★ (combinatorial search + theory), **open-ended**. **Effort:** high. **Risk:** high — fuzzy success criteria. **Language:** Python.
- **Impact:** ★★★★★ if a clean result — genuinely novel, thesis-worthy.
- **Verdict:** The "high risk, high reward" option. *Research update:* full detail in `research/06-dimensionality-reduction.md` (incl. the professor's own project page). Key insight: best k-subset = maximize purity of the truth-table partition (majority per block); error oracle is O(2^n). Scope: exhaustive + greedy + information-theoretic, then test the decomposition-correlation conjecture. Highest novelty, fuzzy success criteria.

### 10. Algebraic factorization
- **What you build:** Turn an SOP into a compact multilevel form via kernel/cube extraction.
- **The algorithmic heart:** Compute kernels & co-kernels, build the co-kernel–cube matrix, find **prime rectangles** (maximal bicliques), greedy-extract and iterate [Bra87]; alternative via graph partitioning [GM99].
- **Depth:** ★★★★★ (dense algebra + matrix/rectangle covering). **Effort:** high. **Risk:** high (essentially reimplementing part of SIS). **Language:** Python.
- **Impact:** ★★★★★ — real synthesis machinery, deeply algorithmic.
- **Verdict:** Hardest listed project. *Research update:* full detail (algebraic division, kernel/co-kernel theory, recursive kernel computation, rectangle covering, single/multiple-cube extraction, GM99 graph partitioning, ABC `fx`) in `research/05-algebraic-factorization.md`. Key insight: factorization reduces to **prime-rectangle covering** in a cube×variable matrix; scope as "good factorization" not optimal.

### 11. DL for circuit-size prediction
- **What you build:** Generate random n-variable functions (n≈10), synthesize with Yosys/ABC, record # of 2-input gates; train a model mapping truth table → gate count.
- **The algorithmic heart:** Data-generation pipeline + feature engineering + regression. **Note:** algorithmic component is thinner — mostly ML + tool automation (may be scrutinized against the "significant algorithmic component" requirement).
- **Depth:** ★★★ (more ML than algorithms). **Effort:** medium-high. **Risk:** medium-high — the core research question (does truth table predict gate count?) may disappoint. **Language:** Python + Yosys/ABC.
- **Impact:** ★★★★★ novelty; flagged as possible master-thesis basis.
- **Verdict:** Great for novelty/ML, but add an algorithmic layer (e.g., spectral/Walsh feature engineering). *Research update:* full detail (data pipeline, SOP-matrix-as-image CNN, Walsh-spectral baseline, GNN, ML-for-EDA SOTA) in `research/10-dl-prediction.md` (incl. the professor's own project page). Key insight: complexity prediction is a *ranking* oracle for decomposition; must add an algorithmic layer to satisfy the course requirement.

---

## Custom proposals (off-list)

| Idea | Algorithmic core | Impact | Realistic |
|---|---|---|---|
| **HLS scheduling** (list scheduling + ILP) | Priority list scheduling; ILP for latency/area | ★★★★ (ties to HLS slides) | Yes |
| **FPGA technology mapping** (FlowMap) | Min-height K-feasible cut enumeration, DP | ★★★★★ (real industry algo) | Yes |
| **Static timing analysis engine** | Longest-path on timing graph, cycle-breaking, slack/arrival | ★★★★ (deterministic, clean) | Yes |
| **SAT-based equivalence checking** | Build CNF for miter circuit, solve | ★★★★ (modeling skill) | Yes |
| **Clock tree synthesis** | Tree balancing to minimize skew | ★★★★ (visual) | Yes |

---

## Risk evaluation & ranking rationale

### What "risk" means here

**Risk = probability you stall or cannot produce a defensible result — *not* difficulty.** A hard but well-specified problem can be low-risk; an easy but ill-defined one can be high-risk. Four factors drive the ratings:

1. **Crispness of "done"** — is there an objective, checkable success criterion?
2. **Published-algorithm vs. must-invent** — reimplementing a known recipe is safer than designing your own.
3. **Number of subtle failure points** — steps where a small bug silently corrupts results (rather than crashing).
4. **Verifiability** — can you *tell* it's correct (visually, or provably via a solver/oracle)?

### Per-project risk justification

**Low risk — #1, #8.**
Both have a hard, objective "done" state. #8's exact minimizer either produces the minimum cover or it doesn't (verifiable against PyEDA/Espresso as an oracle). #1 always yields *a* drawing with instant visual feedback — you can always ship something, and quality is a spectrum, not a pass/fail cliff. No hidden failure modes.

**Low-medium — #5.**
Simulated annealing is forgiving: it *always* produces a valid placement and monotonically improves with cooling. HPWL is trivial to compute and easy to unit-test. There is no state in which you have *nothing* — worst case is a mediocre-but-valid result, and the SA-vs-EO comparison still yields a defensible experimental study.

**Medium — #2, #3.**
Both are complete, published algorithms (Wong-Liu; sequence-pair) — no invention required — and both are **deterministic and testable**: no overlaps + correct bounding-box area = correct geometry. The residual risk is a few fiddly sub-steps where a bug *doesn't crash but silently gives wrong coordinates*: shape-curve arithmetic (#2) and constraint-graph → longest-path extraction (#3). These are debug-able but time-consuming, hence medium rather than low.

**Medium — #7 (SAT/ILP variant).**
Routing *sounds* hard, but the SAT/ILP approach inverts the risk: you **declare constraints** (each net connected, edges don't cross, within grid) and let z3/or-tools *find* the routes. This deletes the genuinely hard part of classic routing — rip-up-and-reroute heuristics that get stuck in local minima or accidentally short nets. Correctness becomes **provable** (solver returns a model or UNSAT). So despite being a "hard" EDA problem, the declarative route has low implementation risk. (The maze-routing variant *would* be medium-high — precisely why the solver variant is recommended.)

**Medium-high — #6, #11.**
- **#6:** the objective (minimize wirelength *subject to edge capacity*) is NP-hard in a way you *feel*: you may never reach a fully **legal** solution and end up presenting "overflow = 3" as your result. Defensible, but weaker than a provably-correct output.
- **#11:** the risk is not the code — it's that the *research question itself* ("does the truth table predict gate count?") may have a disappointing answer, plus a Yosys/ABC automation dependency. A negative/weak result is still presentable, but the payoff is uncertain.

**High — #4-analytical, #9, #10.** Three different reasons:
- **#4-analytical:** legalization (quadratic solution → overlap-free, grid-snapped placement) is *the* classic stall point, and it needs a sparse linear solver on top. Hard to get a clean end-to-end result.
- **#9:** no known efficient algorithm for "best k-variable subset," so you're doing exhaustive search or inventing a heuristic — and there is **no ground truth** to check against. The "explain *why*" part is genuinely open. High variance: a clean finding is great, a muddled one is a weak presentation.
- **#10:** kernel/co-kernel extraction + rectangle covering is a *lot* of surface area (many sub-steps, each with subtle algebra), with no crisp "done" and no easy oracle to verify correctness. Easy to sink 40h into a half-working version.

### Why the ranking came out this way

The ranking is tuned to the stated preference — *interesting + high impact first, but realistic fallback*. That is why high-impact-but-high-risk items (#9, #10, #4-analytical) are **not** at the top, even though they'd rank first under a pure "max impressiveness" lens.

The composite optimized is roughly **impact ÷ risk ÷ effort**:

| # | Project | Impact | Risk | Effort | Why it lands here |
|---|---|---|---|---|---|
| 1 | #7 Detailed routing (SAT/ILP) | ★★★★★ | medium | medium | Provable correctness + visual + modeling wow, at low implementation risk. Best return per hour. |
| 2 | #3 Sequence pairs | ★★★★★ | medium | med-high | Strongest *algorithmic* story, published recipe, testable output. High impact with bounded risk. |
| 3 | #5 SA vs EO placement | ★★★★ | low-med | medium | Not the flashiest, but lowest variance — essentially can't fail, and the comparison adds genuine insight. |
| 4 | #6 Global routing | ★★★★★ | med-high | high | Same impact class as #7 but higher risk (legality) and more effort — so it drops a spot. |
| 5 | #9 Dim. reduction | ★★★★★ | high | high | Highest ceiling, but only if you accept the open-ended risk — hence last, flagged "only if you want research." |

Left off the top-5: **#1, #2, #8** are safe but lower-impact (impact was prioritized); **#10, #4-analytical, #11** are high-impact but the risk/effort isn't justified when #7 and #3 deliver similar impact more cheaply.

**The three-way tradeoff (pick your optimization target):**
- Optimize for **impressive + high effort budget** → the top of the list shifts to #10 or #9.
- Optimize for **guaranteed success with minimal stress** → shifts to #8 or #5.
- Optimize for **impact under a realistic budget** → the ranking above (#7, #3, #5).

**Caveat on "impact":** the impact scores are a judgment of how a *presentation lands* and how well it showcases *algorithmic* skill — not real-world usefulness. If "impact" were redefined as novelty/publishability, the top would instead be #9 and #11.

---

## Bottom line

Ranked against "interesting + impact + realistic":

1. **#7 Detailed routing (SAT/ILP variant)** — best interest-to-effort ratio, provably-correct output.
2. **#3 Sequence-pair floorplanning** — strongest algorithmic story in the course core.
3. **#5 Cell placement (SA vs EO)** — safest balanced pick, still genuinely interesting.
4. **#6 Global routing** — harder graph-optimization challenge.
5. **#9 Dimensionality reduction** — thesis-level novelty, accept the risk.

**Pragmatic strategy:** pick a safe core (#5 or #7) and, if time permits, add a comparison or extension to push toward "interesting" without betting everything on risk.

### Post-research synthesis (validated)

After full-depth research on all 11, the ranking holds, with two refinements:

- **#4 (global placement)** rises when judged by "algorithms current EDA tools use" — the analytical/quadratic lineage (→ ePlace/RePlAce/DREAMPlace) is *the* modern EDA algorithm. But the full nonlinear version is grad-level; the realistic scope is **quadratic placement** (Laplacian + CG + anchor spreading + Tetris legalizer).
- **#7's SAT/ILP route is confirmed** as the highest interest-to-effort (provable legality via multi-commodity-flow); **#6's NCR-vs-ILP** is literally the "algorithm vs. model" the proposal asks for.

**Two lenses on the ranking:**

*Balanced (impact ÷ risk ÷ effort):* #7 > #3 > #5 > #6 > #9.

*Your stated taste (challenge + novelty + real-tool relevance):* #4 (quadratic/analytical) > #3 (sequence pair) > #6 (NCR global routing) > #7 (SAT/ILP detailed routing) > #10 (algebraic factorization) > #9 (dim-reduction novelty) > #11 (ML novelty) > #5/#8/#2/#1 (safe).

---

## References

- [BP03] Boettcher & Percus — Extremal Optimization (2003)
- [Bra87] Brayton — Factoring logic functions (IBM JRD, 1987)
- [Bre77] Breuer — A class of min-cut placement algorithms (DAC, 1977)
- [CPGM14] Cortadella et al. — Boolean Rule-Based Cell Routing (IEEE TCAD, 2014)
- [GM99] Golumbic & Mintz — Factoring logic functions using graph partitioning (ICCAD, 1999)
- [Hu06] Hu — Efficient, high-quality force-directed graph drawing (2006)
- [KGJA91] Kleinhans et al. — GORDIAN (IEEE TCAD, 1991)
- [KGV83] Kirkpatrick, Gelatt, Vecchi — Optimization by simulated annealing (Science, 1983)
- [KK89] Kamada & Kawai — An algorithm for drawing general undirected graphs (1989)
- [Kor05] Koren — Drawing graphs by eigenvectors (2005)
- [McC56] McCluskey — Minimization of boolean functions (BSTJ, 1956)
- [MFNK96] Murata et al. — Sequence-pair (IEEE TCAD, 1996)
- [WL86] Wong & Liu — A new algorithm for floorplan design (DAC, 1986)

Tools: SpyDrNet, PyEDA, KaHyPar, Yosys, NetworkX.

---

## Research dossiers

Full-depth dossiers (problem → idea → algorithm overview → detail → SOTA → refs) live in `project/research/`. Progress:

- ✅ `01-global-placement.md` (#4) — min-cut, quadratic/CG, force-directed, SA, modern nonlinear (ePlace/RePlAce/DREAMPlace), legalization.
- ✅ `02-detailed-routing.md` (#7) — Lee/BFS, Dijkstra, A*, constraint graphs + left-edge/dogleg, SAT/ILP encoding, TritonRoute/Dr. CU.
- ✅ `03-global-routing.md` (#6) — RSMT/Hanan grid, Dijkstra/A*, ILP route-selection, rip-up & reroute, negotiated congestion, FastRoute/NTHU-Route.
- ✅ `04-sequence-pair-floorplanning.md` (#3) — sequence-pair rules, constraint graphs, weighted-LCS decoding, SA move set, B*-tree/O-tree/TCG comparison.
- ✅ `05-algebraic-factorization.md` (#10) — algebraic division, kernel/co-kernel theory, recursive kernels, rectangle covering, single/multiple-cube extraction, GM99, ABC fx.
- ✅ `06-dimensionality-reduction.md` (#9) — subset-purity error oracle, exhaustive/greedy/information-theoretic search, Ashenhurst–Curtis link, professor's project page.
- ✅ `07-cell-placement.md` (#5) — HPWL, simulated annealing (move/swap + cooling), extremal optimization (fitness + τ-EO), avalanche dynamics.
- ✅ `08-two-level-minimization.md` (#8) — QM prime generation, prime-table reductions, Petrick, branch-and-bound, Espresso.
- ✅ `09-slicing-floorplanning.md` (#2) — slicing tree + Polish expression, shape functions + H/V composition, Stockmeyer sizing, Wong–Liu SA moves.
- ✅ `10-dl-prediction.md` (#11) — data pipeline, SOP-matrix CNN, Walsh-spectral baseline, GNN, ML-for-EDA SOTA.
- ✅ `11-graph-drawing.md` (#1) — spectral (Laplacian/Fiedler), Fruchterman–Reingold + multilevel, Kamada–Kawai/stress majorization.
