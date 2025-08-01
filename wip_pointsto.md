In computer science, pointer analysis, or points-to analysis, is a static code analysis technique that establishes which pointers, 
or heap references, can point to which variables, or storage locations. This is the most common colloquial use of the term.
A secondary use has pointer analysis be the collective name for both points-to analysis, defined as above, and alias analysis. 
Points-to and alias analysis are closely related but not always equivalent problems. 

Alias analysis is a technique in compiler theory, used to determine if a storage location may be accessed in more than one way. 
Two pointers are said to be aliased if they point to the same location. 

It exist multiple type of analysis, context insensitive or sensitive, flow sensitive or insensitive.
Basically, flow sensitive approach means that instead of over-approximate the whole function/program we knows the
order of the program statements and computes a distinct points-to solution at each program points. Which is way more precise.

In a context insensitive (monovariant) analysis alls to `foo()` are merged into a single summary. That means if one call passes
it pointer to `x` and another passes it pointer to `y` the analysis will assume inside `foo` that its parameters may points both to x and y.

In context sensitive approach (polyvariant) analysis, you distinguish each call site (or each abstract call-chain up to length k) as a 
separate variant of `foo` so the invocation of `foo(&x)` is analyzed in its own context and `foo(&y)` in another. Inside `foo` the parameter's points-to set for the variaants contains only `x` and for the second only `y`.

# Hardekopof & Lin algorithm
First runs a cheap, flow‑insensitive analysis (AUX) to find out which loads and stores could possibly define or use each heap or address‑taken variable. Giving us a conservative def-use graph of only the relevant pointer operations. Because `AUX` is flow-insensitive, it over-approximates but it's fast and scales to millions of lines.

Unlike prior “semi‑sparse” schemes that only sparsify the top‑level (register) variables, Hardekopf & Lin build SSA form for every pointer variable (even those in memory) and then create a def‑use graph (DUG) whose nodes are only ALLOC, LOAD, STORE, COPY, and φ/χ/µ operations. No other CFG nodes survive. 

Once you have that DUG, you simply propagate points‑to sets along def‑use edges rather than along every edge of the full control‑flow graph. This is the classic “sparse dataflow” trick: you only push information to the nodes that actually need it, and you respect program order because def‑use edges already encode the causal flow of pointer values 

---

## 1. Preprocessing: build the Sparse Def‑Use Graph (DUG)

1. **AUX pass (flow‑insensitive)**

   * Quickly over‑approximates which stores (defs) might reach which loads (uses).
   * Uses that to resolve indirect calls and build an interprocedural CFG (ICFG).
   * Rewrites each call into a set of *COPY* instructions for parameters, and each return into a *COPY* for the return value.

2. **Top‑level SSA & partitioning**

   * Convert all *top‑level* (non‑heap) variables into SSA form, turning each φ‑function into a multi‑input COPY node (e.g. `x₁ = φ(x₂,x₃)` → `x₁ = x₂ x₃`).
   * Group your *address‑taken* (heap/pointer‑loaded) variables into **equivalence classes** (partitions) so that you know which pools of heap cells you’ll track together.

3. **Label loads and stores**

   * For each partition $P$, annotate every STORE that *might* modify $P$ with a χ‑function (`P = χ(P)`), and every LOAD that *might* access $P$ with a μ‑function (`v = μ(P)`).
   * These χ/μ annotations will later guide where you need φ‑nodes in the heap‐SSA.

4. **Heap‑SSA on partitions**

   * Run an SSA transformation on each partition (in effect: place φ‑nodes for χ/μ).
   * Now every heap‐related def or use is in strict SSA form.

5. **Construct the DUG**

   * **Nodes**: one for every ALLOC, COPY, LOAD, STORE, and SSA φ/χ/μ.
   * **Edges**:

     * **Unlabeled** for straight data‑flow on top‑level vars: ALLOC/COPY/LOAD → any use of the same top‑level var.
     * **Labeled** for heap partitions: a STORE with χ on partition $P$ → any φ/χ/μ or LOAD labeled with the same $P$.
     * **Unlabeled** for φ nodes on partitions: φ → any use of that partition var.

---

## 2. Data‑structures for the sparse solver

* **Worklist** `Worklist` ← all ALLOC nodes.
* **Global points‑to graph** `PG` holds the points‑to set for every top‑level variable $v$: $P_{\text{top}}(v)$.
* **Per‑node IN/OUT graphs**

  * For each LOAD or φ node $k$, an `INₖ` graph $Pₖ(v)$ for all address‑taken $v$ reaching that node.
  * For each STORE node $k$, two graphs: `INₖ` (incoming) and `OUTₖ` (outgoing) for all address‑taken $v$ it might write.

All these graphs start empty.

---

## 3. The sparse fix‑point loop

```
while Worklist ≠ ∅:
    k ← removeOne(Worklist)

    switch (kind of node k):
      case ALLOC:
        let x = &o   ⇒  Δ = {(x→o)}
        PG[x] ∪= {o}
        propagate Δ along all (unlabeled) outgoing DUG edges from k

      case COPY:
        let x = y z …  ⇒  Δ = PG[y] ∪ PG[z] ∪ …
        PG[x] ∪= Δ
        propagate Δ along all (unlabeled) outgoing edges from k

      case LOAD (v = μ(P)):
        INₖ[P] ← ⋃ₚart(P) PG[p]    (gather incoming PG info)
        Δ = INₖ[P]                   (the new points-to facts)
        propagate Δ along all (labeled with P) outgoing edges

      case STORE (P = χ(P)):
        INₖ[P] ← ⋃ₚart(P) PG[p]      (gather incoming PG info)
        OUTₖ[P] = INₖ[P] ∖ {old stored targets} ∪ {new stores}
        Δ = OUTₖ[P] ∖ INₖ[P]         (what changed)
        propagate Δ along all (labeled with P) outgoing edges

      case φ (on partition P):
        INₖ[P] ← union of IN-edges’ IN/OUT for P
        Δ = INₖ[P]
        propagate Δ along all (unlabeled) outgoing edges
```

* **Propagation** means: for each target node $m$ reached by some edge from $k$:

  1. Add the new points‑to facts Δ into that node’s IN or `PG` (depending on unlabeled vs. labeled edges).
  2. If that target’s IN (or `PG`) actually grew, re‑add $m$ to `Worklist`.

When the worklist finally empties, every node’s IN/OUT and the global `PG` are at a **fix‑point**, and you have your full, flow‑sensitive points‑to solution.

---

### Why it’s both **flow‑sensitive** and **sparse**

* **Flow‑sensitive**: We respect the order encoded in the DUG edges (which in turn mirror control‑flow plus SSA’s def‑use ordering), so each node sees exactly the pointer facts that reach it *at that program point*.
* **Sparse**: We only ever touch the tiny DUG (pointer ops + SSA φ/χ/μ), not the full CFG.  No unnecessary propagation through non‑pointer statements.

This staged, sparse approach is what lets Hardekopf & Lin scale precise flow‑sensitive analysis to millions of lines of code.

Chase et al [5] propose that SSA form be dynamically computed during the course of the flow-sensitive pointer analysis
Chase et al do not evaluate their idea, but a similar idea is evaluated by Tok et al [31], whose algorithm can analyze
C programs with almost 70,000 lines of code in roughly 30 minutes. The scalability of this approach is limited by the cost
of dynamically updating SSA form.
