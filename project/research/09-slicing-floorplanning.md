# Dossier #2 — Floorplanning with Slicing Structures

> Full-depth research dossier. Sources: Kahng et al. *VLSI Physical Design* §3.3–3.5, Wong & Liu [WL86], Otten/Stockmeyer sizing.

---

## 1. Problem statement

**Input**

- A set of $n$ rectangular **blocks**, each with a set of feasible (width, height) **shape functions** (e.g., rotatable, or a discrete library of orientations).

**Output**

- A **sliceable floorplan** — a placement where the chip is recursively cut by full horizontal/vertical lines — representable as a **binary slicing tree** (or equivalently a normalized Polish expression).
- Final block dimensions + positions.

**Objective (minimize)**

1. **Bounding-box area** (primary).
2. **Total wirelength** (secondary; needs a netlist).

**Constraint**

- The floorplan must be **sliceable** (this distinguishes #2 from #3's general sequence-pair packing). Blocks may be **rotated** per their shape functions.

**Why it's hard.** Minimizing area for slicing floorplans is **polynomial** (shape-function calculus, Stockmeyer) — that's the elegant part — but *choosing which slicing tree* is best is still a search over exponentially many trees; adding wirelength makes it NP-hard. For *non*-slicing floorplans, even area minimization is NP-hard.

---

## 2. Core idea

**Encode the floorplan as a slicing tree**, then compute the optimal area **bottom-up** using **shape functions**:

- Each block has a **shape function** $h(w)$ = minimum height achievable for width $w$.
- **Compose** two blocks (sub-floorplans) by H or V:
  - **Vertical** (stack): $h_F(w) = h_a(w) + h_b(w)$, $w_F = \max(w_a, w_b)$.
  - **Horizontal** (side by side): $w_F(h) = w_a(h) + w_b(h)$, $h_F = \max(h_a, h_b)$.
- The minimum area lies at a **corner point** of the top-level shape function; **backtrack** top-down to fix each block's dimensions.

Then use **simulated annealing over slicing trees (Polish expressions)** to find the best tree for a given cost (area + wirelength).

---

## 3. Algorithms overview & comparison (representations)

| Representation | Structure | Area optimization | Scope |
|---|---|---|---|
| **Slicing tree / Polish expression** (this project) | ordered binary tree (H/V nodes) | **exact in poly time** (Stockmeyer) | slicing only |
| Sequence pair (project #3) | two permutations | longest-path/LCS packing | general (all non-overlap) |
| B*-tree / O-tree | ordered tree | O(n) packing | compact subset |
| TCG | DAG 4-tuple | O(n²) | general |

**Key trade-off:** slicing structures are the *simplest* and admit **exact polynomial area optimization** — but they are a strict *subset* of floorplans (the "wheel" is the smallest non-slicing case). Sequence pair (project #3) is general but loses the clean exact-area calculus. Slicing is thus the natural **first** floorplanning project.

---

## 4. Algorithm detail

### 4.1 Slicing tree and Polish expression

- A **slicing tree** is an ordered binary tree: leaves = blocks, internal nodes = cut type (H or V); child order gives relative position.
- A **Polish expression** is the postorder traversal of the slicing tree, e.g. `12H3V4H` (operands = block IDs, operators = H/V).
- A **normalized** Polish expression has no two consecutive identical operators (e.g., no `HH` or `VV`) — giving a bijection with slicing trees and a compact move space for SA.

### 4.2 Shape functions and composition

For a block with area $A$: $h(w) = A/w$ (with lower bounds or discrete orientations giving a stair-shaped frontier of **corner points**).

Composition of two sub-floorplans $a,b$:
- **Vertical (stack):** $h_F(w)=h_a(w)+h_b(w)$, width $=\max(w_a,w_b)$ — add heights at each width, merge corner points.
- **Horizontal (side-by-side):** $w_F(h)=w_a(h)+w_b(h)$, height $=\max(h_a,h_b)$.

### 4.3 Floorplan sizing (Stockmeyer / Otten)

```
FloorplanSizing(tree, blocks):
  # bottom-up
  for leaf block: h_block(w) = shape function (corner points)
  for internal node (H or V), postorder:
    if V: h_F(w) = h_left(w) + h_right(w)     # merge corner points
    if H: w_F(h) = w_left(h) + w_right(h)
  # top-level optimum
  (w*, h*) = corner point of h_root minimizing w*h
  # top-down backtrack
  assign each block its (w,h) from the chosen corner point, recurse
```

- **Complexity:** polynomial in the number of blocks and corner points (each composition merges two sorted corner-point lists).
- The **minimum area is at a corner point** of the top-level shape function; backtracking fixes individual dimensions.

### 4.4 Search over slicing trees (SA)

Since *which* tree is optimal is unknown, search the space of normalized Polish expressions by simulated annealing.

**Move set** (Wong–Liu):
1. **Swap two adjacent operands** (blocks).
2. **Complement an operator chain** (H↔V) between two operands.
3. **Swap an adjacent operand and operator** (with normalization).

**Cost function:** $cost = \alpha \cdot \text{Area} + \beta \cdot \text{Wirelength} + \gamma \cdot \text{AspectRatioPenalty}$; area comes from §4.3, wirelength from a netlist (HPWL).

```
SA_Slicing(blocks):
  P = random normalized Polish expression
  while T > Tmin:
    while not equilibrium:
      P' = PERTURB(P)             # swap operands / complement chain
      (area, wl) = SIZE_AND_PLACE(P')   # Stockmeyer sizing + HPWL
      Δ = COST(P') - COST(P)
      if Δ < 0 or rand() < exp(-Δ/T): P = P'
    T = α·T
```

---

## 5. State of the art

- **Slicing + shape functions** (Otten 1983, Stockmeyer 1983, Wong–Liu 1986) is a classic, elegant result — exact polynomial area optimization within a fixed slicing tree.
- **Modern practice** has moved to **general (non-slicing)** representations — sequence pair, B*-tree, TCG — because slicing excludes good layouts (the wheel). Slicing is now mainly pedagogical and for regular structures.
- The **shape-function / corner-point calculus** survives conceptually in analytical placers' handling of macro shapes (feasible-region / sizing ideas).

---

## 6. Key references

- [WL86] Wong & Liu, *A new algorithm for floorplan design*, DAC 1986 (the project's cited paper — normalized Polish expressions + SA).
- Otten, *Efficient floorplan optimization*, ICCD 1983 (shape-function composition).
- Stockmeyer, *Optimal orientations of cells in slicing floorplan designs*, Information and Control 1983.
- Guo, Cheng, Yoshimura, *O-tree*, DAC 1999; Chang et al., *B*-trees*, DAC 2000 (for contrast).
- Textbook: Kahng et al., *VLSI Physical Design*, §3.3–3.5.

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★★ — the shape-function/corner-point calculus is genuinely elegant; SA search over Polish expressions adds a real combinatorial layer.
- **Effort:** medium-high — shape-function composition (merging corner-point lists) is fiddly but bounded; SA is straightforward.
- **Risk:** medium — deterministic and testable (verify area correctness, no overlaps), but shape-curve arithmetic is a classic silent-bug source.
- **Course fit:** strong — maps to §3 and the "Floorplanning" block; produces clean visual layouts.

**Practical project scope (if chosen):** implement slicing-tree → Polish-expression parsing, Stockmeyer sizing with rotatable blocks, and SA search (Wong–Liu moves) with area + HPWL cost; verify against a brute-force/enumerated small case, and optionally contrast with the sequence-pair method (project #3) to show what slicing misses.
