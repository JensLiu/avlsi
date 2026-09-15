# Dossier #3 — Floorplanning with Sequence Pairs

> Full-depth research dossier. Sources: Kahng et al. *VLSI Physical Design* §3.4–3.5, Ozdal floorplanning slides, Murata et al. [MFNK96], Tang & Wong (fast evaluation).

---

## 1. Problem statement

**Input**

- A set of $n$ rectangular **blocks** (modules), each with given width/height (possibly rotatable: swap width↔height).
- Optionally: a netlist connecting block pins (for wirelength cost).

**Output**

- A non-overlapping placement of all blocks in the plane (a **floorplan**) — positions $(x_i, y_i)$ and orientations.

**Objective (minimize)**

1. **Area** of the bounding box, $W \times H$.
2. **Total wirelength** (e.g., HPWL of the netlist).

**Constraint on the representation**

- The floorplan must be expressible as a **sequence pair** $(S_+, S_-)$ — two permutations of the block names — *and* the search must operate on this representation (this distinguishes project #3 from #2's slicing trees).

**Why it's hard.** Rectangle packing is NP-hard; the space of floorplans is enormous ($(n!)^2$ sequence pairs). A sequence pair is a *compact encoding* of geometric relations that lets a metaheuristic (simulated annealing) explore non-overlapping layouts efficiently.

---

## 2. Core idea

A **sequence pair** is two permutations $(S_+, S_-)$ of the block names that fully determines the *relative* order (left-of / above) of **every** pair of blocks:

- $a$ **left of** $b$  ⇔  $a$ before $b$ in *both* $S_+$ and $S_-$.
- $a$ **above** $b$  ⇔  $a$ before $b$ in $S_+$ but after $b$ in $S_-$.

These relations become a pair of constraint graphs whose **longest paths give the packed coordinates** — decoding a sequence pair is equivalent to **weighted longest-common-subsequence (LCS)**. The search then perturbs the two permutations (SA) and re-evaluates in $O(n\log n)$.

**The beauty:** every sequence pair decodes to a *legal, non-overlapping* floorplan — so SA never has to repair overlaps; it just optimizes area/wirelength over a huge but well-behaved space.

---

## 3. Algorithms overview & comparison (representations)

| Representation | Structure | Packing cost | Completeness | Notes |
|---|---|---|---|---|
| **Slicing tree** (project #2) | binary tree / Polish expression | O(n) | slicing floorplans only | simple, but misses non-slicing |
| **Sequence pair** (this project) | two permutations | O(n²) naive, O(n log n) fast | all non-overlapping (general) | the classic, [MFNK96] |
| **B*-tree** | ordered binary tree | O(n) | compact subset (left-bottom compact) | fast, popular in practice |
| **O-tree** | rooted ordered tree | O(n) | compact subset | compact encoding |
| **TCG (transitive closure graph)** | 4-tuple of DAGs | O(n²) | general | finer moves, more complex |
| **CBL (corner block list)** | 3-tuples | O(n) | general (mosaic) | less used |

**Key trade-off:** sequence pair is the *simplest representation that captures **all** non-overlapping floorplans* (not just slicing ones), which is its historical significance. B*-tree/O-tree are faster ($O(n)$ packing) but encode only a *subset* of floorplans. TCG is general + has a complete local move set, but is more complex. Sequence pair is the sweet spot for a course project.

---

## 4. Algorithm detail

### 4.1 The sequence-pair encoding

For blocks $a,b$:
- $(S_+: \ldots a \ldots b \ldots, \; S_-: \ldots a \ldots b \ldots)$ ⇒ $a$ is **left of** $b$.
- $(S_+: \ldots a \ldots b \ldots, \; S_-: \ldots b \ldots a \ldots)$ ⇒ $a$ is **above** $b$.

These two rules assign a unique horizontal/vertical relation to *every* pair (hence no overlaps possible after packing).

### 4.2 Floorplan → constraint graphs → sequence pair

1. **HCG** (horizontal constraint graph): node weight = block width; edge $v_i\to v_j$ if $m_i$ is left of $m_j$. **VCG**: node weight = height; edge if $m_i$ is below $m_j$. Remove transitive (redundant) edges.
2. **Longest path** in HCG = minimum width; in VCG = minimum height.
3. Emit $S_+$/$S_-$ by topological order of the two graphs.

### 4.3 Sequence pair → floorplan (evaluation)

Coordinates are obtained by **weighted LCS**: $x = LCS(S_+, S_-,\ \text{widths})$ and $y = LCS(S_+^R, S_-,\ \text{heights})$ (where $S_+^R$ is $S_+$ reversed).

```
SequencePairEval(<S+, S->, widths, heights):
  weights = widths
  (x, W) = LCS(S+, S-, weights)         # x-coords + total width W
  weights = heights
  SR = reverse(S+)
  (y, H) = LCS(SR, S-, weights)         # y-coords + total height H
  return (x, y, W, H)
```

**Weighted LCS** (the $O(n^2)$ textbook version):

```
LCS(S1, S2, weights):
  for i in 1..n: block_order[S2[i]] = i
  lengths[1..n] = 0
  for i in 1..n:
    block = S1[i]; index = block_order[block]
    positions[block] = lengths[index]          # place after blocks packed so far
    span = positions[block] + weights[block]
    for j in index..n:
      if span > lengths[j]: lengths[j] = span
      else: break
  return positions, lengths[n]                  # lengths[n] = total span
```

- **Complexity:** naive $O(n^2)$; Tang & Wong's **fast evaluation** reaches $O(n\log^2 n)$ (and later $O(n\log n)$) by using balanced data structures instead of the inner update loop — important because SA calls evaluation *millions* of times.
- Equivalent formulation: build HCG/VCG explicitly and take longest paths (topological sort + DP), $O(n^2)$ edges worst-case.

### 4.4 Simulated annealing over sequence pairs

**Move set** (perturbations), from [MFNK96]:
1. **Rotate** a block (swap width↔height).
2. **Swap** two blocks in $S_+$ only.
3. **Swap** two blocks in $S_-$ only.
4. **Swap** two blocks in *both* $S_+$ and $S_-$ (a "move" that changes both relations).

**Cost function:** $cost = \alpha\cdot\text{Area} + \beta\cdot\text{Wirelength} + \gamma\cdot\text{AspectRatioPenalty}$ (e.g., penalize aspect ratio far from a target), with weights tuned.

```
SA_SequencePair(blocks):
  S = random initial (S+, S-); T = T0
  (x,y,W,H) = eval(S); cost = COST(W,H,wirelength)
  while T > Tmin:
    while not equilibrium:
      S' = PERTURB(S)          # rotate / swap in S+ / S- / both
      (x',y',W',H') = eval(S')
      Δ = COST(S') - COST(S)
      if Δ < 0 or rand() < exp(-Δ/T): S = S'
    T = α·T
```

- **Decoding is the hot path** — use the fast $O(n\log n)$ evaluation, not the $O(n^2)$ one.
- Perturbation example (Ozdal): swapping two blocks in *both* sequences can dramatically change the packing (e.g., bounding box from 12×8 to 7×12).

---

## 5. State of the art

- **Sequence pair** [MFNK96] and **fast evaluation** [Tang & Wong, ASP-DAC 2001] are the classic results; the representation is still taught and used for mixed-size floorplanning.
- **Modern practice** has largely moved to **B\*-tree** and **O-tree** (faster $O(n)$ packing, compact encodings) and **TCG** (general + complete move set) for block-level floorplanning; and to **analytical placers** (§4 dossier) for standard-cell placement.
- For **mixed-size / macro placement**, hybrid approaches combine B\*-tree/sequence-pair for macros with analytic spreading for standard cells.
- The *algorithmic idea* — a compact encoding whose decoding is a longest-path/LCS, searched by SA — is the lasting lesson, and it generalizes to modern representations.

---

## 6. Key references

- [MFNK96] Murata, Fujiyoshi, Nakatake, Kajitani, *VLSI module placement based on rectangle-packing by the sequence-pair*, IEEE TCAD 1996 (the project's cited paper).
- Tang, Tian, Wong, *Fast evaluation of sequence pair in block placement by longest common subsequence computation*, ASP-DAC 2001.
- Wong & Liu, *A new algorithm for floorplan design*, DAC 1986 (slicing, for contrast with project #2).
- Guo, Cheng, Yoshimura, *An O-tree representation of non-slicing floorplan*, DAC 1999.
- Chang et al., *B\*-trees: a new representation for non-slicing floorplans*, DAC 2000.
- Lin & Chang, *TCG: a transitive closure graph-based representation*, DAC 2001.
- Wong, *On simulated annealing in EDA*, ISPD 2012 (tutorial).
- Textbook: Kahng et al., *VLSI Physical Design*, §3.4–3.5.

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★★½ — the sequence-pair↔floorplan correspondence (constraint graphs + weighted LCS) is genuinely elegant and non-trivial; SA search on top.
- **Effort:** medium-high. The $O(n^2)$ LCS decode is easy; the fast $O(n\log n)$ decode and a tuned SA add real work.
- **Risk:** medium — deterministic and testable (verify no overlaps + area correctness), but the longest-path/LCS coordinate extraction is a classic silent-bug source.
- **Course fit:** strong — maps to §3.4–3.5 and the "Floorplanning" syllabus block; visual layouts make a great presentation.

**Practical project scope (if chosen):** implement sequence-pair → constraint-graph → longest-path decoding, plus SA search with the standard move set and area/wirelength cost; compare against a slicing-tree baseline (ties nicely to project #2). Optional: fast $O(n\log n)$ evaluation, and block rotation.
