# Dossier #6 — Global Routing

> Full-depth research dossier. Sources: Kahng et al. *VLSI Physical Design* §5, Ozdal global-routing slides, and SOTA (FastRoute, NTHU-Route, NCR).

---

## 1. Problem statement

**Input**

- A **routing grid** $G$ (W×H cells); adjacent cells joined by **edges**, each edge has a **capacity** $\sigma(e)$ (max wires that may cross it).
- A set of **terminals** placed on grid cells; terminals are **colored**, and all terminals of one color form one **net** that must be electrically connected.

**Output**

- A set of **wires** (grid-edge paths / Steiner trees) connecting all terminals of each net.

**Constraints**

- **Capacity:** total number of wires using each edge ≤ $\sigma(e)$ (the PDF example uses capacity 2).
- All terminals of each color connected (a connected subgraph per net).

**Objective (minimize)**

- **Total wirelength** (sum of edge usages over all nets), often plus via/bend counts.

**Why it's hard.** Even routing a *single* multi-pin net optimally is the **rectilinear Steiner tree (RSMT)** problem — NP-hard in general. Routing *all* nets under shared edge capacities is a **global congestion** problem (essentially multi-commodity flow / Steiner forest with capacities) — strongly NP-hard. Hence: exact ILP for small instances, and iterative heuristics (rip-up & reroute, negotiated congestion) for real designs.

> This is the coarse-grain step that precedes detailed routing (§7 dossier): decide *which* g-cells each net passes through, leaving exact track assignment for later.

---

## 2. Core idea

Decompose into two levels:

1. **Per-net topology:** connect one net's terminals with a low-cost tree — a **rectilinear Steiner tree (RSMT)** (allow new Steiner branch points) or, simpler, a **rectilinear minimum spanning tree (RMST)**.
2. **Global interaction:** many nets compete for limited edge capacity. Solve by either (a) **declarative ILP** (select legal, non-overflow routes), or (b) **iterative rip-up & reroute / negotiated congestion** (route everything, then punish congested edges and reroute violators until legal).

The central insight of modern global routing is **negotiated congestion**: don't forbid overflow immediately — *price* it, and let nets iteratively negotiate away from congested edges.

---

## 3. Algorithms overview & comparison

| Approach | Mechanism | Optimal? | Scalability | Notes |
|---|---|---|---|---|
| RMST (Prim/Kruskal on Manhattan distances) | spanning tree, no Steiner points | no (≥ RSMT) | easy | simple baseline |
| RSMT via Hanan grid | Steiner points on Hanan grid | optimal small nets | medium | NP-hard general |
| Sequential Steiner heuristic | greedy closest-pair + MBB merging | heuristic (optimal ≤4 pins) | fast | textbook algorithm |
| Maze routing (Dijkstra/A*) | shortest path in weighted grid | optimal single path | medium | per-net / repair |
| ILP global routing | route-option selection + capacity LP | exact (given options) | small–medium | Sidewinder, BoxRouter |
| Rip-up & reroute (RRR) | route all (allow overflow), rip violators, reroute | heuristic | large | industrial workhorse |
| Negotiated congestion (NCR) | RRR + congestion-priced edge costs | heuristic | large | FastRoute, NTHU-Route |

**Key trade-off:** RSMT/RMST handle *topology*; ILP handles *legality* but not scale; RRR/NCR handle *scale + congestion* but are heuristic. Modern routers combine all three (FLUTE for topology, NCR for congestion, A* for reroute).

---

## 4. Algorithm detail

### 4.1 Single-net: Rectilinear Steiner tree (RSMT)

**Idea.** Connect all pins with horizontal/vertical segments; adding extra **Steiner points** (junctions not at a pin) reduces total length. An RSMT for $p$ pins has $0\dots p{-}2$ Steiner points, each of degree 3 or 4.

**Hanan grid theorem:** an RSMT always exists whose Steiner points lie on the grid formed by drawing horizontal/vertical lines through every pin (the **Hanan grid**) — so search only over $O(p^2)$ candidate points, not continuous space.

**Sequential Steiner heuristic** (textbook §5.6.1): greedy Prim-like growth using minimum bounding boxes (MBB).

```
SequentialSteiner(P):
  P' = P
  (pA,pB) = CLOSEST_PAIR(P')          # rectilinear distance
  add pA,pB to tree T; remove from P'
  if P' empty: add L-shape(pA,pB); return T
  curr_MBB = MBB(pA,pB)
  while P' not empty:
    (pMBB', pC') = CLOSEST_PAIR(curr_MBB, P')   # closest between box and a pin
    add pC' to T; remove from P'
    if pMBB' is a pin: add L-shape(pMBB, pC)
    else: add pMBB' as Steiner point; add L-shape containing pMBB'
    curr_MBB = MBB(pMBB', pC')
  add L-shape connecting last pair
```

- Optimal for ≤4 pins; heuristic for more. Standard production alternative: **FLUTE** (fast lookup-table RSMT, used by FastRoute/BoxRouter).

### 4.2 Shortest path (Dijkstra / A*) — maze routing

Used to route a net *within* the routing graph with weighted edges (cost = length + congestion + via penalty). See dossier #7 (§4.2) for full pseudocode. Key role in global routing: **reroute** a ripped-up net along a min-cost path.

### 4.3 ILP global routing (exact, small instances)

**Idea.** Pre-generate a small set of candidate routes per net (L-shapes, Z-shapes, U-shapes); binary variables select one route per net; capacity constraints forbid overflow.

**Variables:** $x_{net,k}\in\{0,1\}$ = net $net$ uses route option $k$ (with desirability weight $w_{net,k}$).

**Constraints:**
- **Mutual exclusion:** $\sum_k x_{net,k} \le 1$ for each net.
- **Capacity:** for each edge $e$, $\sum_{\text{routes using } e} x \le \sigma(e)$.

**Objective:** maximize $\sum_{net,k} w_{net,k} x_{net,k}$ (maximize routed nets / quality; some nets may stay unrouted → fall back to maze routing).

```
ILP_GlobalRoute(G, capacities, Netlist):
  for each net: generate route options (L/Z/U shapes, FLUTE decomposition)
  build ILP: x ∈ {0,1}, sum_k x_netk ≤ 1, capacity constraints
  solve (GLPK/MOSEK/CP-SAT)
  for unrouted nets: maze-route (Dijkstra/A*) 
```

**Tools:** Sidewinder and BoxRouter are ILP-based; both decompose multi-pin nets with FLUTE and iterate (add maze routes, re-solve) until convergence.

### 4.4 Rip-up & reroute (RRR)

**Idea.** Route *all* nets while **allowing temporary violations**, then iteratively rip up violating nets and reroute them so congestion resolves globally (rather than the last net being forced to detour).

```
RRR(Netlist, G):
  v_nets = []
  for each net: ROUTE(net,G)               # allow overflow
    if HAS_VIOLATION(net,G): append v_nets
  while v_nets not empty and not timeout:
    v_nets = REORDER(v_nets)               # optional
    for net in v_nets:
      if HAS_VIOLATION(net,G):
        RIP_UP(net,G); ROUTE(net,G)
        if HAS_VIOLATION(net,G): append v_nets
      remove first element
```

- Key: allowing overflow during initial routing lets *short* nets keep short routes and forces *which nets detour* to be decided by congestion, not arrival order.
- Net **ordering** matters (affects quality); NCR (below) largely removes this sensitivity.

### 4.5 Negotiated congestion routing (NCR)

**Idea.** Make RRR self-correcting by pricing edges by congestion. Each edge has cost $cost(e)$; a net pays $\sum_{e\in net} cost(e)$. Congestion $\phi(e)=\eta(e)/\sigma(e)$ (usage/capacity). After each iteration, **increase** $cost(e)$ for congested edges ($\phi>1$), leave others unchanged, then reroute all nets (Dijkstra/A*) under the new costs.

- Cost **never decreases** (otherwise previously-penalized nets bounce back).
- Growth rate $\Delta cost(e)$ is critical: too fast → nets oscillate between edges; too slow → many iterations. Modeled as linear, logistic, or slowly-growing exponential.
- Effect: nets "negotiate" — critical nets stay on the good edges, others detour — and net ordering becomes less important.

**Pattern routing** (used inside NCR): restrict candidate shapes to **L/Z/U** patterns (Fig. 5.22) for speed; a net that can't be routed by patterns falls back to maze routing. Pattern routing is $O(n)$ vs. maze routing $O(n^2 \log n)$.

---

## 5. State of the art

- **FastRoute 4.0 (2009):** NCR-based; optimizes **via count** throughout the flow (via-aware Steiner), not just in the maze cost. Foundational for modern NCR routers.
- **NTHU-Route 2.0 (2008/2010):** ISPD-2008 contest winner (best on 11/16 benchmarks). Improvements: (1) history-based cost function, (2) better congested-region identification + rip-up/reroute ordering. Fast, stable, robust.
- **BoxRouter 2.0:** box expansion + ILP + maze post-processing.
- **SPRoute (ICCAD 2019):** scalable *parallel* negotiation-based router (multithreaded NCR).
- **Common recipe (modern):** FLUTE (RSMT topology) → pattern routing (L/Z/U) → maze routing (A*) → NCR with history cost → via optimization.

**Open issues:** congestion/routability prediction, via minimization, and handling new design rules; parallelism (SPRoute) is a current frontier.

---

## 6. Key references

- Kahng/Lienig/Markov/Hu, *VLSI Physical Design*, §5 (global routing) — Steiner trees, maze/A*, ILP, RRR, NCR.
- Hanan, *On Steiner's problem with rectilinear distance*, 1966 (Hanan grid).
- Chu & Wong, *FLUTE: fast lookup table based rectilinear Steiner minimal tree algorithm*, IEEE TCAD 2008.
- Dijkstra (1959); Hart–Nilsson–Raphael (A*, 1968).
- Pan & Chu, *FastRoute 4.0*, ICCAD 2009.
- Chang et al., *NTHU-Route 2.0*, IEEE TCAD 2010 (ISPD-2008 winner).
- Cho et al., *BoxRouter 2.0*, ICCAD 2007.
- McMurchie & Ebeling, *PathFinder* (negotiated congestion), FPGA 1995 — the origin of NCR.

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★★½ — Steiner trees (NP-hard), shortest paths, and the genuinely subtle negotiated-congestion pricing loop.
- **Effort:** high. An ILP route (route-option selection + capacity constraints + or-tools) is tractable; a full NCR router (Steiner + A* + history costs) is a substantial but very satisfying build.
- **Risk:** medium-high — reaching a *fully legal* (zero-overflow) solution is NP-hard; you may present "overflow = k" results. The ILP variant guarantees legality for small grids.
- **Course fit:** strong — maps to §5 and the "Global routing" syllabus block; the colored-points + capacity example is directly the project spec.

**Practical project scope (if chosen):** implement (a) an **ILP global router** (FLUTE/Steiner route options + capacity constraints) for small grids and (b) a **NCR heuristic** (pattern routing + A* + congestion-priced costs) for larger ones; compare wirelength, overflow, and runtime. The "mathematical model vs. algorithm" comparison is exactly what the proposal asks for.
