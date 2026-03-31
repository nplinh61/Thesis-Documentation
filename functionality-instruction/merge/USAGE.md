# Model Consistency & Conflict Resolution Analysis

This document details the step-by-step process of managing model consistency rules (Reactions) and resolving potential conflicts using dependency graphs and topological sorting.

---

## 1. Setup

### Models
* **M1**: Component model, containing a **Component** element.
* **M2**: Namespace model, containing a **Namespace** element.

### Consistency Rule (Reaction)
> Whenever `Component.id` changes in **M1**, automatically rename the corresponding `Namespace.id` in **M2** to match.

### Ancestor State (Common Base)
* **M1**: Component `[uuid-C]` | `id = "payment"`
* **M2**: Namespace `[uuid-N]` | `id = "payment"`

---

## 2. Diverging Work

### Engineer A (feature-rename)
* **Commit A1**: Renames `Component.id` from `"payment"` to `"billing"` in **M1**.
* **Reaction**: Automatically sets `Namespace.id` to `"billing"` in **M2**.
* **Changelog (A1.changelog.json)**:
    * `originalChanges`: `[(uuid-C, id, from="payment", to="billing")]`
    * `consequentialFootprint`: `[(uuid-N, id)]`

### Engineer B (feature-namespace)
* **Commit B1**: Directly sets `Namespace.id` to `"payment-gateway"` in **M2**.
* **Reaction**: None (direct change, not via component rename).
* **Changelog (B1.changelog.json)**:
    * `originalChanges`: `[(uuid-N, id, from="payment", to="payment-gateway")]`
    * `consequentialFootprint`: `[]`

---

## 3. Merge Execution Steps

### Step 1 - Find Merge-Base
JGit identifies the common ancestor commit: `abc1234`.
* **Ancestor State**: `Component.id="payment"`, `Namespace.id="payment"`

### Step 2 - Load Changelog DTOs
Collect diverging commits and read JSON files from Git object store:
* `changesA`: `[(uuid-C, id, to="billing")]`
* `changesB`: `[(uuid-N, id, to="payment-gateway")]`
* `consequentialFootprintA1`: `{(uuid-N, id)}`
* `consequentialFootprintB1`: `{}`

### Step 3 - Direct Conflict Detection
Compare `(elementUuid, feature)` pairs:
* **A1** touches: `(uuid-C, id)`
* **B1** touches: `(uuid-N, id)`
* **Result**: `uuid-C ≠ uuid-N` -> Different elements -> **No direct conflict**.

### Step 4 - Resolution of Direct Conflicts
No conflicts to resolve. Proceed.

### Step 5 - Build Dependency Graph
Add inter-branch edges using footprints:

* **Check A1 -> B1**:
    * `consequentialFootprint(A1)` = `{(uuid-N, id)}`
    * `originalFootprint(B1)` = `{(uuid-N, id)}`
    * **Overlap detected ->** Add edge: **A1 -> B1**
    * *Reason*: B1's original change must be the LAST write to `Namespace.id` so B's intent wins.

### Step 6 - Detect Cycles
* **Graph**: `A1 -> B1` (One direction only, no cycle). Proceed.

### Step 7 - Topological Sort (Kahn's Algorithm)
1.  **In-degree**: A1=0, B1=1.
2.  Queue: `[A1]`.
3.  Emit **A1**, reduce B1 in-degree to 0.
4.  Emit **B1**.
* **Interleaving**: `[A1, B1]`

### Step 8 - Replay in Order
**Start state**: `Component.id="payment"`, `Namespace.id="payment"`

1.  **Replay A1**:
    * Apply original change: `Component.id = "billing"`
    * **Reaction fires**: `Namespace.id = "billing"`
    * *State*: `Component.id="billing"`, `Namespace.id="billing"`

2.  **Replay B1**:
    * Apply original change: `Namespace.id = "payment-gateway"`
    * *State*: `Component.id="billing"`, `Namespace.id="payment-gateway"`

### Step 9 - Iterative Refinement
Compare actual footprints against stored estimates:
* **A1**: Stored matches Actual.
* **B1**: Stored matches Actual.
* No new edges -> No restart needed.

---

## 4. Final Result

* **Component.id** = `"billing"` (Engineer A's intent preserved)
* **Namespace.id** = `"payment-gateway"` (Engineer B's intent preserved)

