# Conflict Detection Analysis

This document outlines the setup, step-by-step execution, and final results of the `detectConflicts` logic between two branches.

---

## 1. Setup - Inputs to `detectConflicts`

### **changesA** (Branch A entries since ancestor)
| ID | UUID | Feature | Type | To (Value) | Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **a1** | `unknown` | `name` | `ATTRIBUTE_CHANGED` | `"X"` | Deleted before UUID resolution |
| **a2** | `uuid-42` | `name` | `ATTRIBUTE_CHANGED` | `"EngineBrakeke"` | commit1 |
| **a3** | `uuid-42` | `name` | `ATTRIBUTE_CHANGED` | `"EngineBraker"` | commit2 (newer) |
| **a4** | `uuid-99` | `null` | `ELEMENT_DELETED` | `null` | |
| **a5** | `uuid-55` | `color` | `ATTRIBUTE_CHANGED` | `"red"` | |
| **a6** | `uuid-77` | `name` | `ATTRIBUTE_CHANGED` | `"Alpha"` | |

### **changesB** (Branch B entries since ancestor)
| ID | UUID | Feature | Type | To (Value) | Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **b1** | `uuid-42` | `name` | `ATTRIBUTE_CHANGED` | `"BrakeEngine"` | |
| **b2** | `uuid-99` | `name` | `ATTRIBUTE_CHANGED` | `"ServiceY"` | |
| **b3** | `uuid-55` | `color` | `ATTRIBUTE_CHANGED` | `"red"` | Match A |
| **b4** | `uuid-77` | `name` | `ATTRIBUTE_CHANGED` | `"Alpha"` | Match A |
| **b5** | `uuid-88` | `age` | `ATTRIBUTE_CHANGED` | `"2"` | Unique to B |

---

## 2. Step-by-Step Execution

### **Step 1**
* **Outer loop picks a1.**
* **Result:** UUID is `"unknown"` -> **Skip**.
* **Why:** Element was deleted before the converter could resolve its UUID (tombstoning problem). Without a UUID, we cannot identify the element.

### **Step 2**
* **Outer loop picks a2** (`uuid-42`, `name`, `to=EngineBrakeke`).
* **Inner loop vs changesB:**
    * **vs b1:** UUIDs match. `classifySeverity(a2, b1)`: Both `ATTRIBUTE_CHANGED`, features match, but values differ -> **MEDIUM**.
    * **vs b2–b5:** UUID mismatch -> **Skip**.
* **Best Store:** `{ "uuid-42|name": Conflict(a2, b1, MEDIUM) }`

### **Step 3**
* **Outer loop picks a3** (`uuid-42`, `name`, `to=EngineBraker`).
* **Inner loop vs changesB:**
    * **vs b1:** UUIDs match. `classifySeverity(a3, b1)`: Feature "name" matches, values differ -> **MEDIUM**.
    * **Conflict Resolution:** Existing key `uuid-42|name` is already `MEDIUM`. Since `new.ordinal() > existing.ordinal()` is false, do **NOT** overwrite.
* **Best Store:** `{ "uuid-42|name": Conflict(a2, b1, MEDIUM) }` (unchanged).
  > **Note:** `a2` (the typo commit) remains in best. This illustrates the flaw where the specific commit order affects which change is flagged as the conflict source.

### **Step 4**
* **Outer loop picks a4** (`uuid-99`, `null`, `ELEMENT_DELETED`).
* **Inner loop vs changesB:**
    * **vs b2:** UUIDs match. `classifySeverity(a4, b2)`: One branch deleted, other changed attribute -> **HIGH**.
* **Best Store:** Adds `"uuid-99|null": Conflict(a4, b2, HIGH)`

### **Step 5**
* **Outer loop picks a5** (`uuid-55`, `color`, `to="red"`).
* **Inner loop vs changesB:**
    * **vs b3:** UUIDs match. Both set `color` to `"red"`. Values are identical -> **null** (no conflict).
* **Why:** Both branches independently reached the same state. Auto-resolvable.

### **Step 6**
* **Outer loop picks a6** (`uuid-77`, `name`, `to="Alpha"`).
* **Inner loop vs changesB:**
    * **vs b4:** UUIDs match. Both set `name` to `"Alpha"` -> **null** (no conflict).

### **Step 7**
* **Entry b5** (`uuid-88`) never appears in `changesA`.
* **Result:** Never compared -> **No conflict**.
* **Why:** This element was created/modified only on Branch B. Independent changes to unique elements do not conflict.

---

## 3. Final Results

```json
[
  {
    "elementUuid": "uuid-42",
    "feature": "name",
    "severity": "MEDIUM",
    "changeOnA": "ATTRIBUTE_CHANGED -> EngineBrakeke",
    "changeOnB": "ATTRIBUTE_CHANGED -> BrakeEngine"
  },
  {
    "elementUuid": "uuid-99",
    "feature": null,
    "severity": "HIGH",
    "changeOnA": "ELEMENT_DELETED",
    "changeOnB": "ATTRIBUTE_CHANGED -> ServiceY"
  }
]
```

---

## 4. Summary of Filters

| Check | Filtered Scenario |
| :--- | :--- |
| `uuid == "unknown"` | Deleted element whose UUID was not resolved at capture time |
| `uuid A ≠ uuid B` | Changes to completely different elements |
| `classifySeverity -> null` (both deleted) | Both branches deleted same element - same outcome |
| `classifySeverity -> null` (diff feature) | A changed name, B changed color - independent |
| `classifySeverity -> null` (same value) | Both set `color="red"` - auto-resolvable |
| `severity.ordinal()` not greater | Duplicate (uuid, feature) pair - keeps highest severity only |