# Dossier #1 — Graph Drawing

> Full-depth research dossier. Sources: Koren [Kor05], Hu [Hu06], Kamada–Kawai [KK89], the Graph Drawing handbook, and SOTA (stress majorization, multilevel maxent-stress).

---

## 1. Problem statement

**Input**

- A simple graph $G=(V,E)$ (vertices + edges), optionally weighted.

**Output**

- A 2-D **layout** — coordinates $(x_v, y_v)$ for each vertex — that is **aesthetically pleasing** and **readable**.

**Objective (informal, aesthetic criteria)**

- Short edges (adjacent vertices close), uniform vertex distribution (no crowding), few edge crossings, symmetry preserved, uniform edge lengths.

**Why it's hard.** There is no single "correct" answer — quality is a *set of competing aesthetic heuristics*, not one objective. The standard formulation is an **energy/stress minimization**, which is a non-convex continuous optimization over $2|V|$ variables (non-trivial and iterative), or a **spectral** problem (eigenvectors of the graph Laplacian).

> Note: this is the least EDA-native project; it's generic graph visualization. It's still algorithmically rich (numerical linear algebra + nonlinear optimization), and there is a ready benchmark library.

---

## 2. Core idea

Three canonical families, all aiming at "place connected vertices near each other, spread everything out":

1. **Spectral:** use the smallest **eigenvectors of the Laplacian** as coordinates — a global, deterministic linear-algebra solution.
2. **Force-directed:** simulate physics — attractive forces along edges (springs), repulsive forces between all pairs (electric charges) — and let the system relax to equilibrium.
3. **Kamada–Kawai / stress:** minimize the *stress* = mismatch between graph-theoretic distance (shortest path) and Euclidean distance, over all vertex pairs.

---

## 3. Algorithms overview & comparison

| Algorithm | Objective | Solver | Complexity | Notes |
|---|---|---|---|---|
| **Spectral** (Koren) | maximize $\sum_{ij} w_{ij}\|p_i-p_j\|^2$ s.t. spread | eigenvectors of Laplacian | O(VE) (Lanczos) | global, deterministic, no local minima |
| **Force-directed** (Eades / Fruchterman–Reingold) | spring + repulsion energy | iterative relaxation | O(V²) per iter | simple, aesthetic, local minima |
| **Kamada–Kawai** | stress $\sum_{ij}(d_{ij}-\|p_i-p_j\|)^2$ | Newton / majorization | O(V²) per iter | classic, distance-faithful |
| **Stress majorization** (GKN) | stress | iterative majorization | O(V²) per iter | the modern standard (Graphviz `neato`) |
| **Multilevel force-directed / maxent-stress** (Hu) | stress/entropy | coarsen → solve → refine | near-linear | scales to huge graphs (`sfdp`) |
| **SDE / resistance-distance** | spectral embedding of distances | eigen-decomposition | O(V³)-ish | recent, global |

**Key trade-off:** spectral is fast/global/deterministic but lower aesthetic quality; force-directed is intuitive but slow and local-minima-prone; stress-majorization + multilevel is the modern quality/scalability winner. A project would implement (and compare) 1–3 of these from scratch.

---

## 4. Algorithm detail

### 4.1 Spectral layout (Koren)

**Idea.** Define the weighted Laplacian $L = D - W$ (where $D_{ii}=\sum_j w_{ij}$). The layout that minimizes $\sum_{ij} w_{ij}\|p_i-p_j\|^2$ subject to unit-variance + orthogonality constraints is given by the **smallest non-trivial eigenvectors** of $L$:

- **1-D:** the **Fiedler vector** = eigenvector of the 2nd smallest eigenvalue $\lambda_2$.
- **2-D:** use eigenvectors $u_2, u_3$ of $\lambda_2, \lambda_3$ as x/y coordinates.

```
SpectralLayout(G):
  L = Laplacian(G)                     # L = D - W
  (λ2, u2) = smallest_nonzero_eigenpair(L)   # Fiedler
  (λ3, u3) = next_eigenpair(L)
  for v: (x_v, y_v) = (u2[v], u3[v])   # normalized to [0,1]^2
```

- Compute via **Lanczos/Arnoldi** (sparse) in ~O(VE); no local minima; deterministic. Handles symmetries naturally. Downside: can degenerate (overlapping vertices in dense/high-degree regions).

### 4.2 Force-directed layout (Eades / Fruchterman–Reingold)

**Idea.** Attractive force $f_a(d)=k_a\,d$ along edges (or $d^2/k$ in FR); repulsive force $f_r(d) = k_r/d^2$ between all pairs. Move each vertex by the net force, iteratively, with a **cooling schedule** (displacement capped and decreasing).

```
FruchtermanReingold(G):
  place vertices randomly
  t = initial temperature (area)
  while t > ε:
    for each pair (u,v): compute repulsive force → add to disp[u], disp[v]
    for each edge (u,v): compute attractive force → add to disp[u], disp[v]
    for each v:
      p_v += (disp_v / |disp_v|) * min(|disp_v|, t)   # cap displacement
    t *= cool                                     # cooling
```

- **Grid optimization:** FR approximates repulsion by binning to reach ~O(V) per iteration for large graphs.
- **Multilevel (Hu, `sfdp`):** *coarsen* (contract edge matchings) → lay out coarse graph → *refine* (prolong + local optimization) → recursively. This is what makes force-directed scale to millions of vertices.

### 4.3 Kamada–Kawai and stress majorization

**Idea.** Model each pair of vertices by an "ideal spring" whose rest length = graph-theoretic **shortest-path distance** $d_{ij}$; minimize the **stress**:

$$\text{stress} = \sum_{i<j} w_{ij}\big(d_{ij} - \|p_i - p_j\|\big)^2$$

- **Kamada–Kawai** minimizes this by **Newton's method** (2D, one pair at a time).
- **Stress majorization** (Gansner–Koren–North) replaces Newton with a robust **majorization** step: solve a sequence of convex quadratic bounds, each a sparse linear system — monotonically decreasing stress, no line search.

```
StressMajorization(G):
  compute all-pairs shortest paths d_ij
  initial layout p (e.g., PivotMDS)
  repeat:
    L^w = weighted Laplacian of current layout
    solve L^w p_new = b(p)              # b depends on current distances
    p = p_new                           # majorization: stress non-increasing
  until convergence
```

- This is the basis of Graphviz's `neato`; multilevel **maxent-stress** (GKN) adds an entropy term for uniform density + a multilevel scheme, avoiding the linear solves.

---

## 5. State of the art

- **Stress majorization** (Gansner–Koren–North, 2004) is the de-facto standard for quality.
- **Multilevel maxent-stress** (GKN, 2015) and **`sfdp`** scale to very large graphs.
- **Spectral distance embedding (SDE)** and **resistance-distance stress** (recent preprint) are newer spectral alternatives.
- **Benchmarks:** the "graph drawing library" mentioned in the proposal (e.g., the *Graphviz* / *Rome* / *North* graph collections) provides standard inputs and quality metrics (edge crossings, edge length variance, stress).

---

## 6. Key references

- [Kor05] Koren, *Drawing graphs by eigenvectors: theory and practice*, Computers & Mathematics with Applications 2005.
- [Hu06] Hu, *Efficient, high-quality force-directed graph drawing*, Mathematica Journal 2006 (multilevel).
- [KK89] Kamada & Kawai, *An algorithm for drawing general undirected graphs*, Information Processing Letters 1989.
- Eades, *A heuristic for graph drawing*, Congressus Numerantium 1984.
- Fruchterman & Reingold, *Graph drawing by force-directed placement*, Software: Practice & Experience 1991.
- Gansner, Koren, North, *Graph drawing by stress majorization*, Graph Drawing 2004.
- Battista, Eades, Tamassia, Tollis, *Graph Drawing: Algorithms for the Visualization of Graphs* (handbook).

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★ — real linear algebra (spectral) + nonlinear optimization (force/stress), but well-trodden and non-EDA.
- **Effort:** medium. Spectral is ~30 lines (eigen-solver); force-directed is ~100; Kamada–Kawai/stress a bit more.
- **Risk:** low — you always produce *a* drawing, quality is a spectrum; visual feedback is instant.
- **Course fit:** weak — it's the least EDA-native of all projects; it satisfies the "algorithmic component" but doesn't connect to VLSI concepts. Choose it only if you specifically want a clean, self-contained graph-algorithms project.

**Practical project scope (if chosen):** implement spectral + force-directed (+ optionally Kamada–Kawai) from scratch, evaluate on a standard benchmark library with quality metrics (edge crossings, stress, edge-length variance), and compare runtime vs. quality. Optionally add the multilevel acceleration (Hu) as the "interesting" extension.
