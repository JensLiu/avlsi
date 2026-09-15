# AVLSI — Project Research Plan

## Goal

Thoroughly understand each of the 11 proposed projects (from `projct-proposals.pdf`):

1. The **problem** (formal: input / output / objective / constraints).
2. The **core idea** behind the algorithm(s).
3. The **algorithms themselves** — if several exist: overview → comparison → detailed explanation of each (with pseudocode).
4. **Current state-of-the-art** — what modern EDA tools and recent papers actually do.

The end goal is deep understanding sufficient to implement any chosen project.

## Agreed decisions

- **Depth:** FULL depth for all 11 — pseudocode-level algorithm detail for every project.
- **SOTA scope:** classic/foundational papers + modern (2015–2024) surveys and tools.
- **Order:** prioritized by challenge, novelty, and "algorithms that current EDA tools actually use."

## Research order & rationale

| # | Project | Why it's ranked here |
|---|---|---|
| 1 | #4 Global placement | Analytical placers (ePlace/RePlAce/DREAMPlace) are the crown jewel of modern EDA — hardest + most "real-tool" |
| 2 | #7 Detailed routing | TritonRoute, Dr. CU; maze + SAT/ILP |
| 3 | #6 Global routing | NTHU-Route, FastRoute; negotiated congestion |
| 4 | #3 Sequence-pair floorplanning | Real representation used in placers |
| 5 | #10 Algebraic factorization | ABC kernel extraction — real synthesis |
| 6 | #9 Dimensionality reduction | Novelty |
| 7 | #5 Cell placement | SA/EO metaheuristics |
| 8 | #8 Two-level minimization | QM/Espresso |
| 9 | #2 Slicing floorplanning | Wong-Liu |
| 10 | #11 DL prediction | Novel ML |
| 11 | #1 Graph drawing | Least EDA-native |

## Sources

- **Textbook:** `vlsi-physical-design.pdf` (Kahng, Lienig, Markov, Hu, 2nd ed.) — Ch.2 partitioning, Ch.3 floorplanning, Ch.4 placement, Ch.5 global routing, Ch.6 detailed routing, Ch.7–8 specialized routing & timing.
- **Course slides:** `slides/phys/*.pptx` (Ozdal: partitioning, floorplanning, placement, global routing, detailed routing, routing topology, network flow, intro); `slides/logic/*.pptx` (two-level exact/heuristic, multi-level, BDD, library binding).
- **Primary papers** (from `projct-proposals.pdf` references): [WL86], [MFNK96], [Bre77], [KGJA91], [KGV83], [BP03], [CPGM14], [McC56], [Bra87], [GM99], [Kor05], [Hu06], [KK89].
- **Web:** surveys + modern tools (DREAMPlace, RePlAce, ePlace, Capo, FastRoute, NCTU-GR/NTHU-Route, Dr. CU, TritonRoute, ABC/MVSIS, Espresso).

## Dossier template (per project)

Each `NN-project-name.md` follows this fixed structure so dossiers are comparable:

1. **Problem statement** — formal input/output/objective/constraints.
2. **Core idea** — the key intuition in plain words.
3. **Algorithms overview** — all known approaches at a glance + a comparison table.
4. **Algorithm detail** — pseudocode-level explanation of each approach (idea → algorithm).
5. **State of the art** — what modern tools/papers actually do.
6. **Key references** — primary papers, surveys, textbook sections.
7. **Difficulty & course fit** — complexity, risk, relevance to the course.

## Output structure

```
project/research/
├── 00-plan.md
├── 01-global-placement.md        (#4)
├── 02-detailed-routing.md        (#7)
├── 03-global-routing.md          (#6)
├── 04-sequence-pair-floorplanning.md (#3)
├── 05-algebraic-factorization.md (#10)
├── 06-dimensionality-reduction.md (#9)
├── 07-cell-placement.md          (#5)
├── 08-two-level-minimization.md  (#8)
├── 09-slicing-floorplanning.md   (#2)
├── 10-dl-prediction.md           (#11)
├── 11-graph-drawing.md           (#1)
└── README.md                     (index + re-ranked summary)
```

`project-comparison.md` (master summary) is kept in sync after each dossier is finished.
