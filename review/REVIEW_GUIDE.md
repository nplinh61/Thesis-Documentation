# Review-Guide: Branch-Aware V-SUM for Vitruvius

**Reviewer(s):** Raziyeh Dehghani, Arne Lange, Martin Armbruster, Thomas Weber

**Author:** Linh Nguyen

**Date:** Friday, 10.04.2026, 10:00-11:00

**Duration:** 60 minutes

**Link to Repo:** https://github.com/nplinh61/Vitruv

**To-be-reviewed branch:** vitruvius_codeReview

**Commit-ID:** 188d8540da1ab6e4bca70cb6da07ef809859eeab


---

## Project Structure

All new code is confined to the `vsum` module of the [Vitruv](https://github.com/vitruv-tools/Vitruv) repository. No other modules were modified.

```
Vitruv/
└── vsum/
    └── src/
        ├── main/
           ├── java/tools/vitruv/framework/vsum/
           │   ├── branch/                         ← new package (branch awareness)
           │   │   ├── BranchAwareVirtualModel.java
           │   │   ├── BranchManager.java
           │   │   ├── CommitManager.java
           │   │   ├── MergeManager.java
           │   │   ├── SemanticConflictDetector.java
           │   │   ├── data/                       ← value/result types
           │   │   │   ├── BranchMetadata.java
           │   │   │   ├── BranchState.java
           │   │   │   ├── CommitOptions.java
           │   │   │   ├── CommitResult.java
           │   │   │   ├── ConflictSeverity.java
           │   │   │   ├── FileOperation.java
           │   │   │   ├── ModelMergeResult.java
           │   │   │   ├── ReplayResult.java
           │   │   │   ├── SemanticConflict.java
           │   │   │   └── ValidationResult.java
           │   │   ├── handler/                    ← Git hook response handlers & watchers
           │   │   │   ├── PostCheckoutHandler.java
           │   │   │   ├── PostCommitHandler.java
           │   │   │   ├── PostMergeHandler.java
           │   │   │   ├── PreCommitHandler.java
           │   │   │   ├── VsumMergeWatcher.java
           │   │   │   ├── VsumPostCommitWatcher.java
           │   │   │   ├── VsumReloadWatcher.java
           │   │   │   └── VsumValidationWatcher.java
           │   │   ├── storage/                    ← semantic changelog I/O
           │   │   │   ├── EChangeToEntryConverter.java
           │   │   │   ├── SemanticChangeBuffer.java
           │   │   │   ├── SemanticChangeEntry.java
           │   │   │   ├── SemanticChangeType.java
           │   │   │   └── SemanticChangelogManager.java
           │   │   └── util/                       ← trigger file abstractions & hook installer
           │   │       ├── AbstractTriggerFile.java
           │   │       ├── GitHookInstaller.java
           │   │       ├── MergeResultFile.java
           │   │       ├── MergeTriggerFile.java
           │   │       ├── PostCommitTriggerFile.java
           │   │       ├── ReloadTriggerFile.java
           │   │       ├── ValidationResultFile.java
           │   │       └── ValidationTriggerFile.java
           │   ├── versioning/                     ← new package (versioning & rollback)
           │   │   ├── VersioningService.java
           │   │   ├── VersioningException.java
           │   │   └── data/
           │   │       ├── RollbackPreview.java
           │   │       ├── RollbackResult.java
           │   │       └── VersionMetadata.java
           │   ├── helper/                         ← existing package
           │   │   └── VsumFileSystemLayout.java   ← extended for per-branch path resolution
           │   └── internal/                       ← existing package
           │       ├── InternalVirtualModel.java   ← added reload(VsumFileSystemLayout)
           │       ├── ModelRepository.java        ← added reload() and reload(layout)
           │       ├── ResourceRepositoryImpl.java ← reload logic implementation
           │       └── VirtualModelImpl.java       ← reload orchestration & listener wiring
           └── resources/
               └── git-hooks/                      ← shell scripts installed into .git/hooks/
                   ├── post-checkout
                   ├── post-commit
                   ├── post-merge
                   └── pre-commit
        
```

**Runtime file layout** produced under the project root by the implementation:

```
.vitruvius/
├── vsum/{branch}/                  per-branch VSUM state
│   ├── uuid.uuid
│   ├── correspondences.correspondence
│   └── consistencymetadata/
├── changelogs/{branch}/
│   ├── json/{shortSha}.json        human-readable semantic changelog
│   └── xmi/{shortSha}-{resource}.xmi
├── branches/{branch}.metadata      branch lifecycle metadata (JSON)
└── versions/{versionId}.metadata   version/tag metadata (JSON)
```

---

## 1. Purpose of this Review

This review covers the implementation added to the existing Vitruvius.
The implementation extends the Vitruvius `VirtualModel` with Git branch awareness: each branch
maintains its own VSUM state, changes are tracked semantically, and branches can be merged with
conflict detection.

---

## 2. High-Level Architecture

The existing Vitruvius VSUM operates on a single model state. This thesis extends it so that each
Git branch has its own VSUM state on the file system, and the in-memory VSUM automatically reflects
whichever branch is currently checked out.

```
Developer / Tool
      |
      v
BranchAwareVirtualModel          (decorator around InternalVirtualModel)
      |  delegates all V-SUM operations
      v
InternalVirtualModel (VirtualModelImpl)
      |  reloaded per branch switch
      v
.vitruvius/vsum/{branch}/        (per-branch storage)
    uuid.uuid
    correspondences.correspondence
    consistencymetadata/
```

Branch switching is triggered either via the Vitruvius API or via Git CLI. Both paths converge on
the same reload mechanism through a file-based IPC channel using trigger files and background
watcher threads.

### New Dependencies

Two external libraries were added to the `vsum` module as part of this implementation:

| Library  | Purpose |
|---|---|
| Eclipse JGit (`org.eclipse.jgit`) |  Programmatic Git operations: branch management, commits, merges, annotated tags, tree walking, and object store access |
| Google Gson (`com.google.code.gson`) |  JSON serialization and deserialization of semantic changelog files and all metadata records (branch, version, merge) |

All other dependencies (`tools.vitruv.change.changederivation` for XMI delta snapshots, Log4j for
logging) were already present in the module before this thesis.

---

## 3. Technical Walkthrough

### 3.1 Git Hook Infrastructure

**Key files:** `post-checkout`, `post-commit`, `post-merge`, `pre-commit` (shell scripts under
`src/main/resources/git-hooks/`), `VsumReloadWatcher`, `VsumPostCommitWatcher`, `VsumMergeWatcher`,
`VsumValidationWatcher`, `PostCheckoutHandler`, `PostCommitHandler`, `PostMergeHandler`, `PreCommitHandler`

Git hooks are the entry point for CLI-triggered operations. They cannot call Java directly, so a
file-based IPC pattern is used throughout:

```
git CLI action
    -> hook writes trigger file under .vitruvius/
    -> background watcher thread (polls every 500ms) detects trigger
    -> watcher calls the appropriate Java handler
```

`VsumReloadWatcher`, `VsumMergeWatcher`, and `VsumValidationWatcher` each use a `FileLock` on their
own `.lock` file (`.reload.lock`, `.merge.lock`, `.validation.lock`) to ensure only one operation
runs at a time. If the lock is held, the trigger is re-written for retry on the next poll cycle so
no events are silently dropped. `VsumPostCommitWatcher` does not use a lock because changelog
writing is idempotent: if the buffer was already drained by `CommitManager`, the watcher attempt
is a no-op.

| Hook | Trigger file | Watcher | Handler |
|---|---|---|---|
| `pre-commit` | `validate-trigger` | `VsumValidationWatcher` | `PreCommitHandler` |
| `post-checkout` | `reload-trigger` | `VsumReloadWatcher` | `PostCheckoutHandler` |
| `post-commit` | `post-commit-trigger` | `VsumPostCommitWatcher` | `PostCommitHandler` |
| `post-merge` | `merge-trigger` | `VsumMergeWatcher` | `PostMergeHandler` |

The shell scripts are installed into `.git/hooks/` at VSUM initialization time.

---

### 3.2 Per-Branch File System Layout

**Key class:** `VsumFileSystemLayout`

The original Vitruvius layout stored VSUM files directly under the project folder. This extension
maps all VSUM files into a per-branch subdirectory:

```
.vitruvius/vsum/{branch}/
    uuid.uuid
    correspondences.correspondence
    consistencymetadata/
```

`VsumFileSystemLayout` resolves the current branch name from JGit HEAD on construction and points
`getUuidsURI()`, `getCorrespondencesURI()`, and `getConsistencyMetadataModelURI()` to the
corresponding per-branch paths.

`inheritFromBranchIfEmpty(sourceBranch)` is called when a branch is first checked out. If the
branch has no `uuid.uuid` yet, it copies `uuid.uuid` and `correspondences.correspondence` from the
parent branch as the starting state. If the files already exist (returning to an existing branch)
this is a no-op.

`forBranch(repoRoot, branchName)` is a factory method used during branch switching to construct a
layout for a known branch name without reading the repository's HEAD ref.

---

### 3.3 Branch-Aware VSUM: Wrapping, Context Switching

**Key classes:** `BranchAwareVirtualModel`, `VsumReloadWatcher`

#### Why Existing Files Were Modified

Branch switching requires swapping the in-memory VSUM state to a different branch's files without
disposing and recreating the model instance. Disposing is not an option because
`VirtualModelRegistry` enforces a one-instance-per-folder constraint: a dispose/recreate would
break the registry and invalidate any references held by the caller. The solution is an in-place
reload, which required targeted additions to five existing files:

| File | Change |
|---|---|
| `VsumFileSystemLayout` | Added per-branch path resolution and the `forBranch()` factory method |
| `InternalVirtualModel` | Added `reload(VsumFileSystemLayout)` to the internal interface |
| `ModelRepository` | Added `reload()` and `reload(VsumFileSystemLayout)` to the repository interface |
| `ResourceRepositoryImpl` | Implemented the reload logic: unloads all resources, resets the UUID resolver, and reloads from disk using the new layout |
| `VirtualModelImpl` | Orchestrates the reload call and notifies registered listeners |

No other parts of the existing Vitruvius codebase were touched.

#### Wrapping

`BranchAwareVirtualModel` implements `InternalVirtualModel` using the decorator pattern. It wraps a
single `VirtualModelImpl` instance and delegates all standard VSUM operations
(`propagateChange`, `createSelector`, `getCorrespondenceModel`, etc.) to it.

The `VirtualModelRegistry` constraint (only one instance per storage folder may be active at a
time) means there is always exactly one active instance, reloaded in place on each branch switch.

#### Context Switching

`switchBranch(oldBranch, newBranch)` performs a branch switch without disposing or recreating the
wrapped model:

1. Constructs a `VsumFileSystemLayout` for the target branch
2. Calls `prepare()` to create the branch directory on disk if needed
3. Calls `inheritFromBranchIfEmpty(oldBranch)` so a new branch starts from the parent's VSUM state
4. Calls `activeModel.reload(newLayout)` to reset the in-memory resource set, UUID resolver, and
   correspondence model to point at the new branch's files
5. Updates `activeBranchName` only after successful reload

`VsumReloadWatcher` calls `switchBranch()` when a `post-checkout` hook trigger is detected. For
API-triggered branch switches, `BranchManager.switchBranch()` calls
`PostCheckoutHandler.onBranchSwitch()` which drives the same reload path.
---

### 3.4 Branch Management

**Key classes:** `BranchManager`, `CommitManager`, `MergeManager`

`BranchManager` is the single owner of all branch lifecycle operations and metadata persistence.

| Operation | Method | Notes |
|---|---|---|
| Create branch | `createBranch(name, parent)` | JGit + write metadata |
| Switch branch | `switchBranch(name)` | JGit checkout + trigger `PostCheckoutHandler` |
| List branches | `listBranches()` | Reads all `.metadata` files |
| Mark as merged | `markAsMerged(name)` | Updates metadata state |
| Ensure metadata exists | `ensureMetadataExists(name, parent)` | Creates metadata for CLI-created branches |

Branch metadata is persisted at `.vitruvius/branches/{branch}.metadata` (JSON). Metadata for
branches created via Git CLI is written by `PostCheckoutHandler` on first checkout via
`ensureMetadataExists()`.

`CommitManager` handles Git commit operations on model files. It auto-stages modified model files
 together with the branch metadata
`lastModified` update. After the commit, if semantic change tracking is attached and model files
changed, it writes the JSON/XMI changelog synchronously and stages the produced files. Finally,
it writes the `post-commit` trigger file so `VsumPostCommitWatcher` can also react to the commit.


`MergeManager` handles Git merge operations between branches. It performs a three-way merge of a
source branch into the current branch via JGit's `RECURSIVE` strategy and maps the result to a
`ModelMergeResult` (`SUCCESS`, `FAST_FORWARD`, `CONFLICTING`, or `FAILED`). On a successful
merge it marks the source branch as `MERGED` and writes the `merge-trigger` file so
`VsumMergeWatcher` can validate and reload the merged VSUM state. On `CONFLICTING` it returns
the list of conflicting file paths. Conflict classification, severity assignment, and resolution
modes are ongoing work covered in Section 3.6.

#### Data Classes

| Class / Enum | Purpose                                                                                  |
|---|------------------------------------------------------------------------------------------|
| `BranchMetadata` | Persisted record: name, `BranchState`, parent branch, created/updated timestamps         |
| `BranchState` | `ACTIVE`, `MERGED`, `DELETED`                                                            |
| `CommitResult` | Commit SHA, branch, author name and email, timestamp, list of staged file paths, flag whether model files changed |
| `CommitOptions` | Staging options: additional files to include and files to exclude from auto-staging       |
| `ValidationResult` | Validation outcome used for both pre-commit and post-merge checks: `hasErrors()`, `hasWarnings()`, list of errors, list of warnings |
| `ModelMergeResult` | Merge outcome: `MergeStatus`, list of conflicting file paths                             |
| `ReplayResult` | Semantic analysis result: ancestor SHA, per-branch change lists, `SemanticConflict` list |
| `SemanticConflict` | Conflicting element UUID, feature name, both change entries, `ConflictSeverity`          |
| `ConflictSeverity` | `HIGH`, `MEDIUM`, `LOW`                                                                  |
| `FileOperation` | `ADDED`, `MODIFIED`, `DELETED`, `RENAMED` (mirrors Git status codes)                     |

---

### 3.5 Semantic Change Tracking

When a commit is made through the Vitruvius API, the framework records not just that files changed
but what changed at the model level.

#### How Changes are Collected

`SemanticChangeBuffer` implements `ChangePropagationListener` and is registered on the
`VirtualModel`. During change propagation, only original (user-initiated) `EChange` instances are
appended to the buffer, keyed by resource URI. Reaction and consequential changes are explicitly
filtered out because they can be re-derived by replaying the originals through Vitruvius. At
commit time, `CommitManager` calls `drainChanges()` to atomically retrieve and clear the buffer.

#### How the Changelog is Written

`SemanticChangelogManager.write()` receives the drained changes and produces the following files per commit:

| File | Location | Purpose |
|---|---|---|
| JSON | `.vitruvius/changelogs/{branch}/json/{shortSha}.json` | Human-readable, operation-based, UUID-keyed |
| XMI | `.vitruvius/changelogs/{branch}/xmi/{shortSha}-{resource}.xmi` | Machine-readable delta snapshot for replay |

`EChangeToEntryConverter` translates each `EChange` into a `SemanticChangeEntry`.
`detectOperation()` classifies each file as `ADDED`, `MODIFIED`, `DELETED`, or `RENAMED` using a
JGit diff against the parent commit as the primary source, with an EChange-based heuristic as
fallback for in-memory-only resources. `oldPath` is populated for `RENAMED` entries.


### 3.6 Merge and Conflict Detection

**Implementation status:** File-level merging is complete. Semantic conflict detection has its
pipeline in place. Conflict categorization, severity classification, and resolution strategies are
ongoing work.

#### Two Merge Entry Points

| Entry point | How it works |
|---|---|
| Via API (`MergeManager.merge()`) | JGit three-way merge, writes merge trigger, `VsumMergeWatcher` validates asynchronously |
| Via Git CLI (`git merge`) | `post-merge` hook writes trigger, `VsumMergeWatcher` picks it up |

In both cases the watcher calls `PostMergeHandler.performPostMerge()`, which runs three steps in
sequence: validates the merged model state, copies the source branch's VSUM directory to the
target branch if validation passes and then reloads the `VirtualModel`, and finally marks the
source branch as `MERGED` regardless of the validation outcome.

#### File-Level Merge (`MergeManager`)

`MergeManager.merge()` delegates to JGit's `RECURSIVE` strategy and maps the result to a
`ModelMergeResult`:

| `MergeStatus` | Meaning |
|---|---|
| `SUCCESS` | Clean three-way merge commit created |
| `FAST_FORWARD` | Source was ahead; HEAD moved forward, no merge commit needed |
| `CONFLICTING` | Conflict markers written into XMI files; developer must resolve manually |
| `FAILED` | Merge could not be attempted |

On `SUCCESS` or `FAST_FORWARD` the source branch is marked `MERGED`. On `CONFLICTING` the list of
affected file paths is captured from JGit and stored in the merge metadata.

#### VSUM State After Merge

`PostMergeHandler` copies the source branch's VSUM directory to the target branch after a
successful merge. This is correct for **fast-forward merges** where the source history is a strict
extension of the target. For three-way merges with diverged history, proper merging of `uuid.uuid`
and `correspondences.correspondence` is explicitly descoped.

#### Semantic Conflict Detection and Replay Pipeline (`SemanticConflictDetector`)

`SemanticConflictDetector.analyzeBranches(branchA, branchB)` implements the full replay-based merge
pipeline. The pipeline has 10 steps. Steps 1–3 are implemented. Steps 4–9 have
stub methods in place with documented TODOs.

**Steps 1–3 (implemented):**

1. Find the merge-base (common ancestor) via JGit `RevFilter.MERGE_BASE`
2. Collect diverging commit SHAs for each branch and read the corresponding JSON changelog files
   directly from the Git object store via `TreeWalk` (no checkout required)
3. Direct conflict detection: compare `(elementUuid, feature)` pairs across both branches;
   identical changes on both sides are not a conflict

Severity classification in step 3:

| Condition | Severity |
|---|---|
| Element deleted on one side, any change on the other | `HIGH` |
| Same reference feature changed to different values | `HIGH` |
| Same attribute feature changed to different values | `MEDIUM` |
| Multi-valued list insertion/removal on same feature | `LOW` |

**Steps 4–9 (pipeline structure in place, not yet implemented):**

4. Resolution of direct conflicts: programmatic or interactive; returns early with unresolved
   conflicts so the caller can handle them before replay proceeds
5. Build footprint dependency graph: nodes per commit, intra-branch ordering edges, inter-branch
   edges where a commit's consequential footprint overlaps another commit's original footprint
6. Cycle detection: a cycle in the dependency graph means no valid interleaving exists
   (consequential conflict, requires human resolution)
7. Topological sort: Kahn's algorithm produces the interleaving order for replay
8. Replay in topological order: deserialize each commit's original changes into live `EChange`
   objects and apply them through the Vitruvius Reaction engine so consequential changes are
   regenerated; guard failures trigger re-ordering
9. Iterative footprint refinement: compare actual consequential footprints captured during replay
   against stored estimates; rebuild and re-sort if new overlaps appear

The result is a `ReplayResult` with per-branch change lists and a deduplicated list of
`SemanticConflict` instances. The severity rules are a first draft and are the current focus of
ongoing work.


---

### 3.7 Versioning

`VersioningService` provides named model snapshots backed by Git annotated tags, with a two-step
rollback flow.

#### Creating and Querying Versions

`createVersion(versionId, description)` creates an annotated Git tag at the current HEAD and writes
a `VersionMetadata` record to `.vitruvius/versions/{versionId}.metadata`. Fields stored: version ID
(matching the tag name), commit SHA, branch, tagger name and email, creation timestamp, description.

`listVersions()` returns all versions sorted newest-first. `getVersion(versionId)` reads a single
entry. `deleteVersion(versionId)` removes both the Git tag and the metadata file.

#### Two-Step Rollback

Rollback is split into preview and confirm to avoid irreversible data loss.

`previewRollback(versionId)` computes the following without modifying anything:
- Commits that will be abandoned (between target and current HEAD)
- Files that will change (via JGit `DiffFormatter`)
- Whether uncommitted changes are present in the working tree

Only when `confirmRollback(preview)` is called does `git reset --hard` execute, followed by a
`VirtualModel.reload()` to reflect the restored state. The returned `RollbackResult` covers three
outcomes: `success`, `successReloadFailed` (reset worked but VSUM reload threw an exception), and
`failed` (the Git reset itself failed).

#### Branching from a Version

`createBranchFromVersion(branchName, versionId)` creates a Git branch at the version's commit,
seeds its VSUM directory by extracting the source branch's VSUM files directly from the commit
tree via JGit `TreeWalk` (no checkout required), and writes a `BranchMetadata` record for the
new branch with state `ACTIVE` and the version's original branch as parent.

