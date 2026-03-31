No problem! Here is the transcription of your **Replay Pipeline** notes in a clean Markdown format for easy sharing or documentation.

---

# Replay Pipeline

### 1. State Extraction & Direct Conflict Detection
* **1. Implemented:** Find merge base: `JGit MERGE_BASE`, already done in Semantic Conflict Detector.
* **2. Partial, TBD:** Store consequential footprint. Load changelog DTOs from Git Object store for both branches: Semantic Conflict Detector (including footprint computation).
* **3. Implemented:** Direct conflict detection.

---

### 2. Dependency Graph Construction & Iterative Refinement
* **4. Planned:** Resolution of direct conflicts: human or programmatic.
* **5. New: Build footprint dependency graph**
    * **Nodes:** one per commit from each branch.
    * **Intra-branch edges:** preserve commit order within each branch.
    * **Inter-branch edges:** If A’s consequential footprint overlaps B’s original footprint $\rightarrow$ **A must be replayed before B**.
* **6. New:** Check for cycles made by consequential conflicts, requires human intervention.
* **7. Planned:** Topological sort $\rightarrow$ interleaving order: **Kahn’s Algo**.
* **8. Replay in topological order**
    * For each commit in order:
        1.  **New:** Check guard: Does target element still exist in current model state?
            * If **NOT**, guard failure (element was deleted by something replayed earlier).
        2.  **New:** Deserialize changelog DTO $\rightarrow$ live `EChange <EObject>` using `HierarchicalId`.
        3.  **Planned:** Apply via `ChangeRecordingView` $\rightarrow$ Reactions fire $\rightarrow$ Consequential changes regenerated.
        4.  **New:** Capture actual consequential footprint.
* **9. New: Iterative Refinement**
    * Compare actual footprint from **Step 8.4** against stored footprints at commit time.
    * If new overlap found $\rightarrow$ add edges, rebuild graph, resort, repeat from **Step 8**.
    * If new cycle $\rightarrow$ consequential conflict.

---

### 3. Conflict Resolution
* **10. Planned:** Result: consistent merged model state, or conflict list for human resolution.