# AVLSI Project Research — Index

Full-depth research on all 11 proposed projects (from `projct-proposals.pdf`). Each dossier follows the same template: **problem → core idea → algorithms (overview → comparison → detail) → state of the art → references → difficulty/course-fit**.

## Dossiers

| File | Project | Core algorithms covered |
|---|---|---|
| `01-global-placement.md` | #4 Global placement | min-cut (FM), quadratic/CG, force-directed, SA, modern nonlinear (ePlace/RePlAce/DREAMPlace), legalization |
| `02-detailed-routing.md` | #7 Detailed routing | Lee/BFS, Dijkstra, A*, HCG/VCG + left-edge/dogleg, SAT/ILP (multi-commodity flow), TritonRoute/Dr. CU |
| `03-global-routing.md` | #6 Global routing | RSMT/Hanan, Dijkstra/A*, ILP route-selection, rip-up & reroute, negotiated congestion, FastRoute/NTHU-Route |
| `04-sequence-pair-floorplanning.md` | #3 Sequence pairs | sequence-pair rules, constraint graphs, weighted-LCS decoding, SA moves, B*-tree/O-tree/TCG |
| `05-algebraic-factorization.md` | #10 Algebraic factorization | algebraic division, kernel/co-kernel theory, recursive kernels, rectangle covering, extraction, ABC `fx` |
| `06-dimensionality-reduction.md` | #9 Dimensionality reduction | subset-purity error oracle, exhaustive/greedy/information-theoretic search, Ashenhurst–Curtis link |
| `07-cell-placement.md` | #5 Cell placement | HPWL, simulated annealing, extremal optimization (fitness + τ-EO) |
| `08-two-level-minimization.md` | #8 Two-level minimization | QM prime generation, prime-table reductions, Petrick, branch-and-bound, Espresso |
| `09-slicing-floorplanning.md` | #2 Slicing floorplanning | slicing tree + Polish expression, shape functions, Stockmeyer sizing, Wong–Liu SA |
| `10-dl-prediction.md` | #11 DL circuit-size prediction | data pipeline, SOP-matrix CNN, Walsh-spectral baseline, GNN, ML-for-EDA |
| `11-graph-drawing.md` | #1 Graph drawing | spectral (Laplacian/Fiedler), Fruchterman–Reingold + multilevel, Kamada–Kawai/stress majorization |

Master summary + ratings: `../project-comparison.md` (kept in sync).

## Source provenance & verification status

- **Verified primary (papers fetched & read):** ePlace (DAC'14), RePlAce (TCAD'18), TritonRoute (ICCAD'18), Dr. CU 2.0 (ICCAD'19), CPGM14 (TCAD'14), Bra87 (IBM JRD'87), BP03 (arXiv abstract); plus the professor's project pages (#9, #11).
- **Verified via textbook/slides (read in full):** the classic algorithms (min-cut/FM, quadratic/CG, maze/A*/Dijkstra, Steiner/Hanan, left-edge/dogleg, sequence-pair/constraint-graph/LCS, slicing/shape-functions, SA, QM/set-cover, kernel theory) — the textbook (Kahng §2–6) and De Micheli/Ozdal slides are authoritative for these.
- **Summary-level (search only; paywalled originals not opened):** DREAMPlace (DAC'19), NTHU-Route 2.0 (TCAD'10), FastRoute 4.0 (ICCAD'09), and the paywalled classics MFNK96 / WL86 / Bre77 / KGJA91 / KGV83 / McC56 / GM99 / Kor05 / Hu06 / KK89 (covered indirectly by the textbook/slides).

**Corrections made during verification:** (1) ePlace/RePlAce use the **weighted-average** wirelength model, not log-sum-exp; (2) [CPGM14] is a **SAT-based standard-cell router** (Euler degree constraints + ILP&LNS), not full-chip detailed routing; (3) ePlace is **flat (FFT density), no netlist clustering**; (4) TritonRoute's initial version is **MILP panel routing**, Dr. CU 2.0 is **design-rule-aware maze routing**.

## Synthesis — what the research changed

1. **The "implement vs. innovate" picture is now precise.** Most projects are *Tier 1/2* (reimplement a published algorithm + experiment); only **#9 and #11** are genuinely open/novel (both have dedicated professor project pages, both flagged as thesis material).

2. **Every project has a "real-tool" anchor** that grounds the SOTA:
   - #4 → ePlace/RePlAce/DREAMPlace (analytical placers)
   - #7 → TritonRoute/Dr. CU
   - #6 → FastRoute/NTHU-Route (negotiated congestion)
   - #3 → sequence-pair lineage (B*-tree/O-tree/TCG)
   - #10 → ABC `fx`
   - #8 → Espresso / Espresso-exact
   - #2 → Wong–Liu / Stockmeyer sizing

3. **Refined scopes (the "don't over-build" corrections):**
   - #4 analytical ≠ full ePlace; the realistic scope is **quadratic placement** (Laplacian + CG + anchors + Tetris).
   - #7/#6 both split cleanly into **"mathematical model (SAT/ILP) vs. algorithm (heuristic)"** — exactly the proposal's wording, and a natural comparison.
   - #10 should be scoped as "good factorization," not "optimal" (no crisp oracle).

## Two rankings

**A. Balanced — impact ÷ risk ÷ effort** (if you want a strong, low-variance result):

1. #7 Detailed routing (SAT/ILP) — provable correctness, visual, best return/hour
2. #3 Sequence-pair floorplanning — strongest algorithmic story
3. #5 Cell placement (SA vs EO) — essentially can't fail
4. #6 Global routing — harder, same impact class as #7
5. #9 Dimensionality reduction — highest ceiling, highest risk

**B. Your stated taste — challenge + novelty + "algorithms current EDA tools use":**

1. #4 Global placement (quadratic/analytical) — *the* modern EDA algorithm
2. #3 Sequence-pair floorplanning — real representation
3. #6 Global routing (NCR) — industrial routers
4. #7 Detailed routing (SAT/ILP) — provable + modern tools
5. #10 Algebraic factorization — real synthesis machinery
6. #9 Dimensionality reduction — novelty/research
7. #11 DL prediction — novelty/ML
8. #5, #8, #2, #1 — safe, lower "wow"

## Recommendation

If you want **challenge + real-tool relevance with a realistic fallback**, the sweet spot is **#4 scoped as quadratic placement** or **#3 sequence-pair floorplanning** — both are core EDA, visually demonstrable, and have a well-bounded implementation path.

If you want **provable, satisfying, lowest-effort-for-the-impact**: **#7 detailed routing (SAT/ILP)**.

If you want **novelty/thesis upside** (and accept fuzzier success): **#9** or **#11**.
