# Dossier #8 — Two-Level Minimization (Quine-McCluskey)

> Full-depth research dossier. Sources: De Micheli *Two-Level Logic Synthesis* slides (DT7/DT8), McCluskey [McC56], Quine, and the Espresso literature.

---

## 1. Problem statement

**Input**

- A Boolean function $f: \{0,1\}^n \to \{0,1\}$, possibly **incompletely specified** (a don't-care set, e.g., the `-` entries in the proposal's truth table).

**Output**

- A **minimum** (or minimal) **sum-of-products (SOP)** representation — a cover of **prime implicants** with the fewest implicants (and, secondarily, fewest literals).

**Objective**

- Minimize #implicants (PLA rows), then #literals (transistors). Exact = *minimum* cover; heuristic = *minimal/irredundant* cover (possibly suboptimal).

**Why it's hard.** Two phases: (1) generate all prime implicants, and (2) select a minimum cover — the **set cover problem**, which is NP-hard. The prime implicant table has up to $2^n$ minterms and $3^n/n$ primes, so the table is exponential.

---

## 2. Core idea

**Quine's theorem:** there exists a *minimum* cover that consists only of **prime implicants**. Hence the problem decouples:

1. **Generate all prime implicants** (Quine-McCluskey iterative combining).
2. **Select the minimum subset** covering all ON-set minterms (set cover: Petrick's method or branch-and-bound).

A **prime implicant** is a maximal cube contained in the ON-set ∪ DC-set; an **essential** prime is the *only* prime covering some minterm (must be in every cover).

---

## 3. Algorithms overview & comparison

| Algorithm | Phase 1 (primes) | Phase 2 (cover) | Exact? | Scalability |
|---|---|---|---|---|
| **Quine-McCluskey** | iterative combining | Petrick / branch-and-bound | exact | small (≤ ~12–15 vars) |
| **Espresso** (heuristic) | EXPAND | IRREDUNDANT | no (minimal) | large |
| **Espresso-exact** | implicit (BDD) | branch-and-bound on cyclic core | exact | most benchmarks |
| **MINI / PRESTO** | various | various | no | medium |

**Key trade-off:** QM is exact but exponential (fine for a course project and small functions); Espresso scales but is heuristic. Modern exact solvers (Espresso-exact) use **BDDs** to represent the implicant table implicitly, making exact minimization tractable for most benchmarks.

---

## 4. Algorithm detail

### 4.1 Phase 1 — generate prime implicants (Quine-McCluskey)

Represent each implicant as a ternary string over $\{0,1,-\}$ (`-` = don't-care literal).

```
QM_primes(minterms, dontcares):
  terms = (minterms ∪ dontcares) as binary strings
  group terms by number of 1s
  primes = ∅
  repeat:
    next = ∅; combined = ∅
    for adjacent groups (g, g+1):
      for each (a in g, b in g+1):
        if a and b differ in exactly one position:
          c = a with that position set to '-'
          next ∪= c; mark a,b as combined
    primes ∪= { terms not combined }
    terms = next (deduped)
  until no new combinations
  return primes
```

- A term that is never combined with any other is a **prime implicant**.
- Combining $a$ and $b$ (differing in one bit) removes that literal (the Boolean identity $x y + x\bar y = x$).

### 4.2 Phase 2 — minimum cover (set cover)

Build the **prime implicant table**: rows = ON-set minterms, columns = primes, entry 1 if prime covers minterm. The task: select the fewest columns covering all rows.

**Reduction rules (preprocessing, applied repeatedly):**
1. **Essential column:** a row with a single 1 → its column is essential; include it, delete its covered rows.
2. **Column (implicant) dominance:** if column $i$'s 1s ⊇ column $j$'s 1s, delete $j$ (dominated).
3. **Row (minterm) dominance:** if row $i$'s 1s ⊇ row $j$'s 1s, delete $i$ (covering $j$ implies covering $i$).
4. **Partitioning:** if the matrix is block-diagonal, solve each block independently.

After reductions, the residual is the **cyclic core** — solve exactly.

**Petrick's method:** write, for each minterm, a clause = OR of the primes covering it; the cover condition is the AND of all clauses. Multiply out (distribute) into SOP, each product term = a cover; pick the smallest. Exact but the multiplication is exponential.

**Branch-and-bound (EXACT_COVER):**

```
EXACT_COVER(A, x, b):              # b = best cover so far
  reduce A (essentials, dominance, partition); update x
  if current_estimate ≥ |b|: return b          # bound
  if A has no rows: return x
  c = branching column                         # e.g., a max-degree column
  x_c = 1
  Ã = A with column c and rows it covers removed
  x' = EXACT_COVER(Ã, x, b)
  if |x'| < |b|: b = x'
  x_c = 0                                       # branch: exclude c
  x'' = EXACT_COVER(A with c removed, x, b)
  if |x''| < |b|: b = x''
  return b
```

- **Bounding function:** lower bound = #selected so far + (max independent set of remaining rows / clique number), since independent rows (no shared column) each need a distinct prime.

### 4.3 Heuristic: Espresso

The classic heuristic minimizer iterates three operators to find a *minimal* (irredundant) cover:

- **EXPAND:** grow each implicant into a prime (greedily flip `-` into 0/1 where legal).
- **REDUCE:** shrink implicants (introduce 0/1 from `-`) to escape local optima.
- **IRREDUNDANT:** drop implicants that are fully covered by others.

```
Espresso(f):
  F = initial cover
  repeat:
    F = EXPAND(F)          # all implicants prime
    F = IRREDUNDANT(F)     # remove redundancy
    F = REDUCE(F)          # perturb to escape local minimum
  until no improvement / time limit
```

- Fast, widely used, not guaranteed optimal; handles multi-output and don't-cares well.

### 4.4 Example (from the proposal)

The proposal's truth table (with don't-cares) should reduce to $f = ab + \bar a \bar c d$ — i.e., two prime implicants selected by the covering phase. The primes $\bar a \bar c d$ and $ab$ cover the ON-set, with the `-` entries (don't-cares) used to enlarge the cubes.

---

## 5. State of the art

- **Exact:** QM's exponential table is mitigated by **implicit BDD-based** methods (Espresso-exact, McGeer–Sanghavi–Brayton), which solve *almost all benchmarks exactly*; heuristic Espresso is used only for speed.
- **Heuristic:** Espresso-MV (multi-valued), MINI, PRESTO.
- **In the modern flow:** two-level minimization is a *local* step inside multi-level tools (ABC/MVSIS), not the main engine — but the underlying **set-cover + branch-and-bound** machinery is the same core used throughout synthesis.

---

## 6. Key references

- Quine, *The problem of simplifying truth functions*, Amer. Math. Monthly 1952.
- [McC56] McCluskey, *Minimization of Boolean functions*, Bell System Tech. J. 1956 (the project's cited paper).
- Petrick, *A direct determination of the irredundant forms of a Boolean function*, 1956.
- Brayton, Hachtel, McMullen, Sangiovanni-Vincentelli, *Logic Minimization Algorithms for VLSI Synthesis* (Espresso), Kluwer 1984.
- McGeer, Sanghavi, Brayton, Sangiovanni-Vincentelli, *Espresso-Signature / exact two-level minimization*, 1993.
- De Micheli, *Synthesis and Optimization of Digital Circuits*, McGraw-Hill 1994.

---

## 7. Difficulty & course fit

- **Algorithmic depth:** ★★★½ — the QM combining is simple; the interesting part is the **set-cover phase** (essentials, dominance, Petrick, branch-and-bound with bounding). Genuine but bounded.
- **Effort:** low-medium. Phase 1 is a few dozen lines; Phase 2 is a clean branch-and-bound.
- **Risk:** low — the "done" state is crisp (verify against PyEDA/Espresso), and there's a clear oracle.
- **Course fit:** strong — maps to the "Two-level logic synthesis" block (De Micheli slides); the proposal's truth-table example is a ready-made test case.

**Practical project scope (if chosen):** implement QM (prime generation + branch-and-bound minimum cover with the reduction rules and don't-care handling); verify against PyEDA; optionally add Espresso's EXPAND/IRREDUNDANT heuristic and compare exact-vs-heuristic on random/benchmark functions.
