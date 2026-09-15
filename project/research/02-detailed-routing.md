# Dossier #7 — Detailed Routing

> Full-depth research dossier. Sources: Kahng et al. *VLSI Physical Design* §5.6 (maze/Dijkstra/A*), §6 (channel/switchbox), Ozdal detailed-routing slides, and SOTA (TritonRoute, Dr. CU).

---

## 1. Problem statement

The project frames the problem explicitly as a **Flow-Free-like** puzzle:

**Input**

- A 2D grid (e.g., 4×5) of routing cells (g-cells); adjacent cells are joined by edges (the routing tracks).
- A set of **terminal pairs** $\{(s_1,t_1), (s_2,t_2), \dots\}$, each pair = one **net** that must be connected by a wire.

**Output**

- For every net, a **path** (sequence of adjacent grid cells) from $s_i$ to $t_i$.

**Constraints**

- **No shorts:** paths of *different* nets may not share any grid edge/vertex.
- Paths stay within the grid and (optionally) obey per-edge **capacity** $c(e)$ (number of wires allowed to cross an edge).
- Optional: minimize **total wirelength** (sum of path lengths) or **number of vias/bends**.

**Why it's hard.** Even the single-net case is shortest-path (easy). The multi-net case ("connect all pairs with non-crossing wires") is NP-hard in general — it's a **multi-commodity flow / vertex-disjoint paths** problem, and minimizing total length subject to disjointness is exactly the hard part. This is why either (a) a global solver (SAT/ILP) or (b) a rip-up-and-reroute heuristic is used.

> Context (verified primary): [CPGM14] is a **SAT-based standard-cell router** — it routes the *internal* wires of library cells (Nangate 45nm) by encoding routability as a SAT formula over a grid graph using **Euler's degree constraints** (parity + "floating terminals"), then optimizes wirelength via **ILP + large-neighborhood search (LNS)**. Project #7 is a *simplified* version: connect pairs of dots on a 2D grid, no shorts (Flow-Free-like), dropping the design rules. So the "SAT/ILP model" option below is exactly in the spirit of [CPGM14]. Full-chip detailed routing (TritonRoute/Dr. CU) is the broader industrial context.

---

## 2. Core idea

There are two fundamentally different mindsets:

1. **Constructive (search):** route nets one at a time with a shortest-path algorithm over the grid (Lee's maze routing / A*), treating previously-routed wires as obstacles; if a net gets blocked, **rip up and reroute**.
2. **Declarative (model + solve):** write the whole problem as a **SAT/ILP** constraint system ("these nets are connected, these edges don't exceed capacity, minimize length") and let a solver *prove* a legal (optimal) solution exists.

Classic EDA also contributes a third, more specialized body: **channel routing** (constraint graphs + left-edge), which is the historical workhorse for two-layer detailed routing.

---

## 3. Algorithms overview & comparison

| Approach | Core mechanism | Completeness | Scalability | Quality | When to use |
|---|---|---|---|---|---|
| Maze routing (Lee / BFS) | wavefront grid expansion, backtrace | optimal for 1 net | small grids | min-length per net | single net, small instance |
| Dijkstra | best-first on weighted grid | optimal (nonneg. weights) | medium | min-cost per net | weighted obstacles/vias |
| A* search | Dijkstra + admissible heuristic | optimal (admissible h) | better than Dijkstra | min-cost per net | guided search toward target |
| Channel routing (left-edge, dogleg) | HCG/VCG + greedy track packing | optimal track count (no VCG cycles) | classic 2-layer | min tracks | legacy channel routing |
| SAT / ILP encoding | multi-commodity flow / CNF constraints | **exact, provably legal** | small–medium | min length (ILP) | the project's "model" option |
| Rip-up & reroute | route + rip blocked nets + iterate | heuristic | large | good | industrial routers |
| Modern: TritonRoute / Dr. CU | sparse grid graph + A*-based path search + DRC repair | heuristic | industrial scale | best-in-class | real designs |

**Key trade-off:** maze/A* are simple and optimal per-net but don't handle global interaction; SAT/ILP gives *provable* global legality but doesn't scale; industrial routers (TritonRoute, Dr. CU) are layered A*-based heuristics with DRC repair. For this project, **SAT/ILP is the highest-return choice** (provable correctness, clean modeling story); **A* + rip-up-reroute** is the more "algorithmic" alternative.

---

## 4. Algorithm detail

### 4.1 Maze routing — Lee's algorithm (BFS)

**Idea.** Flood the grid from the source as a wavefront; when the wave reaches the target, backtrace parent pointers to recover a shortest path. Obstacles (already-routed wires) are simply not expanded.

**Lee's algorithm** (unweighted grid, 4-neighborhood):

```
Lee(s, t, grid):
  for all cells c: visited[c]=false, parent[c]=∅
  queue = {s}; visited[s]=true; dist[s]=0
  while queue not empty:
    u = pop(queue)
    if u == t: break
    for each neighbor v of u (N/E/S/W):
      if in-bounds(v) and not obstacle(v) and not visited[v]:
        visited[v]=true; parent[v]=u; dist[v]=dist[u]+1
        push(queue, v)
  if not visited[t]: return FAIL
  path = []; u = t
  while u != s: path.prepend(u); u = parent[u]
  return [s] + path
```

- **Complexity:** O(V+E) per net (BFS); memory O(V). On a grid, V = cells.
- **Lee's original** is exactly this BFS "wave" (four directions; §7.2.2 adds diagonals = octilinear).
- **Drawbacks:** explores in a circle (slow); memory-heavy for large grids. Used directly only for small instances or single nets.

### 4.2 Dijkstra & A* search (weighted maze routing)

**Idea.** Make it *best-first*: always expand the node with minimal cost-so-far. Dijkstra uses $g(v)$ = cost from source. A* adds an **admissible heuristic** $h(v)$ = estimated cost to target, so priority = $f(v)=g(v)+h(v)$. With an admissible $h$ (never overestimates, e.g., Manhattan distance), A* is optimal and expands far fewer nodes.

**Dijkstra** (three-group formulation):

```
Dijkstra(G, W, s, t):
  cost[s]=0; for v≠s: cost[v]=∞; parent[*]=UNKNOWN
  group1=V; group2=∅; group3=∅
  move s: group1→group3; curr=s
  while curr != t:
    for each neighbor node of curr:
      if node in group3: continue
      trial = cost[curr] + W[curr][node]
      if node in group1: move node→group2; cost[node]=trial; parent[node]=curr
      else if trial < cost[node]: cost[node]=trial; parent[node]=curr
    curr = node in group2 with min cost; move curr→group3
  backtrace from t via parent
```

**A*** = same, but `curr = argmin_{group2}(cost[v] + h(v))` where $h$ is a lower bound on remaining cost (e.g., Manhattan distance to target). Fig. 5.19: A* expands ~⅓ the nodes of Dijkstra. **Bidirectional A*** expands from both s and t.

### 4.3 Channel routing — constraint graphs + left-edge

*(The classic "detailed routing" of the course syllabus; gives theoretical grounding.)*

**Setup.** A *channel* is a horizontal strip with pins on top and bottom (vectors TOP, BOT). Two layers: horizontal tracks + vertical columns.

**Constraint graphs.**
- **Horizontal constraint graph (HCG):** nets $i,j$ connected by an undirected edge if their horizontal spans overlap (⇒ they need different tracks). Equivalently, the **zone representation**: $S(col)$ = nets passing column $col$; maximal $S(col)$ sets give a **lower bound** on tracks = $\max_{col}|S(col)|$.
- **Vertical constraint graph (VCG):** directed edge $i\to j$ if net $i$'s trunk must be *above* net $j$'s (pins in same column). A **cycle** in the VCG = vertical conflict ⇒ resolve by **net splitting (dogleg)**.

**Left-edge algorithm** (Hashimoto–Stevens): greedily pack as many nets as possible onto the topmost track, moving left to right.

```
LeftEdge(CR):
  curr_track = 1
  unassigned = all nets
  while unassigned ≠ ∅:
    VCG = build_VCG(CR); ZR = zone_rep(CR)
    sort unassigned by leftmost start column
    for each net n in that order:
      if VCG_parents(n)==∅ and no conflict(n, curr_track):
        assign n → curr_track; remove n
    curr_track += 1
```

- If the VCG is acyclic, left-edge achieves the **minimum track count** = $\max|S(col)|$.
- **Dogleg routing** (Deutsch): split a $p$-pin net into $p{-}1$ subnets (only at pin columns) to break VCG cycles and reduce tracks, then run left-edge.

**Switchbox routing** generalizes channel routing to pins on all four sides (fixed dimensions); algorithms derive from greedy channel routers (Luk, BEAVER, PACKER).

### 4.4 SAT / ILP encoding (the project's declarative option)

**Idea.** Model the whole multi-net routing as an integer program; the solver *guarantees* no shorts and finds min-length or feasible routes.

**ILP — multi-commodity flow formulation.** Let grid edges $E$ have capacity $c_e$. For each net $k$ with source $s_k$, sink $t_k$:

- Variables: $x^k_e \in \{0,1\}$ for directed edge $e$, indicating net $k$ uses $e$.
- **Flow conservation** (connectivity): for each node $v$,
$$\sum_{e\in\delta^+(v)} x^k_e - \sum_{e\in\delta^-(v)} x^k_e = b^k_v,\quad b^k_v = \begin{cases}+1 & v=s_k \\ -1 & v=t_k \\ 0 & \text{else}\end{cases}$$
- **Capacity / no shorts:** $\sum_k (x^k_{uv}+x^k_{vu}) \le c_e$ for every undirected edge $uv$. (With $c_e=1$ this enforces vertex/edge-disjoint single wires.)
- **Objective:** minimize $\sum_{k,e} x^k_e$ (total wirelength), or add bend/via penalties.

This is a standard **integer multi-commodity flow**; small grids solve exactly with CBC/SCIP/Gurobi/CP-SAT (or-tools). A cheaper *heuristic* is the LP relaxation + randomized rounding.

**SAT encoding alternative.** For each net and each grid cell, a "reaching" predicate; enforce: terminals reached, each non-terminal has ≤2 incident used edges (no branching), no two nets share an edge, and forbid cycles. A SAT solver (CaDiCaL/MiniSat) yields a satisfying assignment = legal routes. (This is precisely [CPGM14]'s approach, verified: routability as a SAT formula with **Euler degree constraints** — each terminal region has exactly one external edge, every intermediate grid vertex has degree 0 or 2 — plus consistency constraints forcing adjacent same-net wires to the same net; [CPGM14] then minimizes wirelength with **ILP + large-neighborhood search**, re-solving one net at a time.)

### 4.5 Modern detailed routers (SOTA)

- **TritonRoute** (Kahng/Wang/Xu; ICCAD 2018 initial router; TCAD 2020 full version in OpenROAD) — *verified primary.* ISPD-2018 contest. Key idea: **intra-layer parallel routing** — split each layer into unit-width panels and route each panel with a **MILP** (maximum-weighted-independent-set on a candidate **conflict graph** capturing shorts + DRC); layers routed **bottom-to-top sequentially**. Preprocessing *splits/merges/bridges* route guides and builds a Steiner-tree guide-to-guide topology; segments realized via **access points (AP)/AP clusters** joined by an MST. Results: up to 74% (avg 50%) raw-score reduction vs. contest winners; up to 93.85% DRC-violation reduction. C++ + CPLEX + OpenMP.
- **Dr. CU / Dr. CU 2.0** (CUHK; ASPDAC 2019 / ICCAD 2019) — *verified primary.* Original Dr. CU: **sparse grid graph + minimum-area-captured path search** (an A*-based cost). Dr. CU 2.0 adds **correct-by-construction** handling of new rules (parallel-run-length, end-of-line-with-parallel-edge, corner-to-corner) via **design-rule-aware maze routing**, **off-track-via** pin access for hard-to-access pins, and **lookup-table-based via type selection**. Flow: access-point assignment → multi-threaded maze routing (rip-up & reroute with guide expansion) → via selection → post-refinement. Results: 2% better than ISPD'19 best; 69% better than Dr. CU on ISPD'18. Source: github.com/cuhk-eda/dr-cu.
- Others: **RegularRoute**, **SmartDR**, **BonnRoute**.

Common thread: all are **layered A*/maze search + DRC repair**, not the old two-layer channel/left-edge approach (which is now textbook/legacy).

---

## 5. State of the art (summary)

- **Single-net:** Lee/BFS (unweighted), Dijkstra (weighted), **A*** with Manhattan/lower-bound heuristics — all standard, all optimal.
- **Multi-net legal routing:** exact via **ILP/SAT** (small–medium instances only); heuristic via **rip-up & reroute** (route one net, rip blocked nets, iterate) — the industrial standard.
- **Industrial:** TritonRoute (MILP panel routing, parallel) and Dr. CU 2.0 (design-rule-aware maze routing) dominate open-source detailed routing; the *fundamental* machinery is still maze/A* + rip-up-reroute, now wrapped in MILP/DRC-aware frameworks.

---

## 6. Key references

- [CPGM14] Cortadella, Petit, Gómez, Moll, *A Boolean Rule-Based Approach for Manufacturability-Aware Cell Routing*, IEEE TCAD 2014 (the project's cited paper).
- Lee, *An algorithm for path connection and its applications*, IRE Trans. EC 1961 (maze routing).
- Dijkstra, *A note on two problems in connexion with graphs*, 1959.
- Hart, Nilsson, Raphael, *A formal basis for the heuristic determination of minimum cost paths* (A*), 1968.
- Hashimoto & Stevens, *Wire routing by optimizing channel assignment* (left-edge), DAC 1971.
- Deutsch, *A dogleg channel router*, DAC 1976.
- Rivest & Fiduccia, *A greedy channel router*, DAC 1982.
- Kahng et al., *TritonRoute*, IEEE TCAD 2020.
- Li et al., *Dr. CU / Dr. CU 2.0*, IEEE TCAD 2019 / ICCAD 2019.
- Textbook: Kahng/Lienig/Markov/Hu, *VLSI Physical Design*, §5.6 and §6.

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★★ — shortest-path/A* (clean, well-understood) + the genuinely interesting multi-net legality (NP-hard, solved exactly by ILP/SAT or heuristically by rip-up-reroute).
- **Effort:** medium. The SAT/ILP route is surprisingly tractable: encode constraints, call or-tools/z3, done. A*-based rip-up-reroute is more code but still bounded.
- **Risk:** medium (SAT/ILP variant) — correctness is *provable*, visual output is unambiguous. The maze-routing variant is medium-high (rip-up-reroute tuning).
- **Course fit:** strong — maps to §6 (detailed routing) + §5.6 (maze/A*), and the "Flow Free" framing makes a very demonstrable presentation.

**Practical project scope (if chosen):** implement a grid detailed router with **two interchangeable engines** — (1) an ILP/SAT encoder (or-tools/z3) and (2) an A* + rip-up-reroute heuristic — then compare legality, wirelength, and runtime across random/generated instances. This is exactly "an algorithm *or* a mathematical model," and comparing both is a natural, impressive extension.
