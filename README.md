## PES-VCS Lab Report
Name: Gagana C


SRN: PES1UG24AM099


Platform: Ubuntu 24.04

---

## Build Instructions

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
export PES_AUTHOR="Gagana C <PES1UG24AM099>"
make all
```

---

## Phase 1 — Object Storage Foundation

Files modified: object.c

object_write prepends a "<type> <size>\0" header to the data, hashes the whole thing with SHA-256, skips writing if the object already exists (deduplication), creates the shard directory, writes to a temp file, fsyncs, and renames atomically. This guarantees the object store is never left with a partial file even on a crash.

object_read reverifies the SHA-256 after reading (integrity check), parses the type and declared size from the header, validates the declared size against the actual byte count, then returns the data portion in a caller-owned buffer.

### Screenshot 1A — ./test_objects

<img width="1786" height="756" alt="image" src="https://github.com/user-attachments/assets/af213e31-94aa-47ca-9aed-6f7b49f253d9" />


### Screenshot 1B — find .pes/objects -type f

<img width="2232" height="752" alt="image" src="https://github.com/user-attachments/assets/de377382-5edd-4ea7-9e1f-798bb9489222" />


---

## Phase 2 — Tree Objects

Files modified: tree.c, Makefile

tree_from_index heap-allocates the Index struct (it is ~5.6 MB, which exceeds the safe stack budget), loads the index, sorts entries by path, then calls the recursive write_tree_level helper.

write_tree_level walks the sorted entries at one directory level. For each plain file entry (no / in the remaining path) it adds a blob TreeEntry. For each directory it groups all entries that share the top-level component, recurses to get the subtree's hash, and adds a 040000 tree entry. After processing all entries it serialises the Tree struct and writes it to the object store.

### Screenshot 2A — ./test_tree

<img width="1912" height="422" alt="image" src="https://github.com/user-attachments/assets/8ebeb6d4-4162-4131-844b-c6542eede7ce" />


### Screenshot 2B — xxd of a raw tree object

<img width="2004" height="1442" alt="image" src="https://github.com/user-attachments/assets/cf53d4e5-482f-4447-a0b9-8349947ade27" />


---

## Phase 3 — The Index (Staging Area)

Files modified: index.c

index_load opens .pes/index with fopen("r") and parses each line with fscanf using the format %o %64s %llu %u %511s. A missing index file is treated as an empty staging area, not an error.

index_save makes a heap-allocated sorted copy (again to avoid a large stack frame), writes to a .tmp file, calls fflush + fsync for durability, then renames atomically.

index_add reads the file, stores it as OBJ_BLOB, stats the file for mtime/size metadata used for fast change detection, upserts the entry, and calls index_save.

### Screenshot 3A — pes init → pes add → pes status

<img width="2246" height="1246" alt="image" src="https://github.com/user-attachments/assets/6eea99d6-3879-4376-a284-cf7faf4d6338" />


### Screenshot 3B — cat .pes/index

<img width="1648" height="192" alt="image" src="https://github.com/user-attachments/assets/f867618a-5670-493a-9008-f4434cd2ab6a" />


---

## Phase 4 — Commits and History

Files modified: commit.c

commit_create calls tree_from_index to snapshot the staged state, reads the current HEAD to find the parent commit (absent for the first commit), fills a Commit struct with author (PES_AUTHOR env var), Unix timestamp, and message, serialises it with commit_serialize, stores it via object_write(OBJ_COMMIT), then calls head_update to advance the branch pointer atomically.

### Screenshot 4A — pes log with three commits

<img width="1292" height="896" alt="image" src="https://github.com/user-attachments/assets/0f0963c5-81b7-47d0-bdb2-213c147d72ac" />


### Screenshot 4B — find .pes -type f | sort

<img width="1736" height="558" alt="image" src="https://github.com/user-attachments/assets/85910de2-8f67-4159-8d6b-e7635abb63aa" />


### Screenshot 4C — Reference chain

<img width="1936" height="334" alt="image" src="https://github.com/user-attachments/assets/c4645644-bc9d-48af-84c8-853add16d1b0" />


---

## Final — make test-integration

Integration test part 1
<img width="1772" height="1312" alt="image" src="https://github.com/user-attachments/assets/6df3f120-f30c-452f-9518-722e5f1e2457" />


Integration test part 2
<img width="1518" height="1314" alt="image" src="https://github.com/user-attachments/assets/770b8823-1499-453e-b5ba-e1c67b2b641f" />


Integration test part 3
<img width="1568" height="1322" alt="image" src="https://github.com/user-attachments/assets/bfd0809f-c639-44f4-ba20-ca8e63b7666d" />


---

## Phase 5 — Branching and Checkout

### Q5.1 — How would you implement pes checkout <branch>?

Files that must change in .pes/:

.pes/HEAD — rewrite the line from ref: refs/heads/<current> to ref: refs/heads/<target>.
If the target branch is new, create .pes/refs/heads/<target> pointing to the desired commit hash.

Working-directory update:

Read the target branch tip from .pes/refs/heads/<branch>.
object_read the root tree of that commit.
Recursively walk the tree: for blob entries write (or overwrite) the file in the working directory; for tree entries (mode 040000) create directories.
Delete any working-directory file that is present in the current HEAD's tree but absent from the target tree.

What makes this complex:

Dirty-file conflicts. If a tracked file has local modifications and the target branch has a different version of that file, checkout must refuse or the user's edits are destroyed silently.

Untracked-file collisions. If an untracked file sits at a path that the target tree would create, checkout must refuse to avoid clobbering it.

Partial failure. A crash halfway through leaves a mixed working directory. A production implementation writes a lock or in-progress marker to allow recovery.

Three-way diff. Deciding which files to touch requires comparing the current-HEAD tree, the target-HEAD tree, and the index all at once.

---

### Q5.2 — Detecting dirty working-directory conflicts

Using only the index and the object store:

Load the current index (the staged snapshot matching HEAD after the last commit).
For each entry, stat() the working-directory file. If st_mtime or st_size differs from the stored values, re-hash the file (SHA-256) and compare to the stored hash. A mismatch means the file is dirty.

For each path in the target branch's tree: if that path is in the index and is dirty, refuse checkout for that path.

For untracked files: if a path in the target tree does not appear in the index but already exists on disk, refuse checkout to avoid overwriting it.

The mtime/size check is a fast first pass; the full hash re-read is the authoritative check used only when metadata differs.

---

### Q5.3 — Detached HEAD and recovery

When HEAD holds a raw commit hash instead of ref: refs/heads/<branch>, head_update writes new hashes directly into HEAD. Commits still chain correctly via parent pointers, but no branch file is updated. Once you check out a branch, HEAD is overwritten and the detached chain has no ref pointing to it — it becomes unreachable from all named references.

Recovery before switching away:

```bash
cat .pes/HEAD
echo <hash> > .pes/refs/heads/recovered
echo "ref: refs/heads/recovered" > .pes/HEAD
```

Recovery after switching away: objects still exist in object store and can be found by scanning commits.

---

## Phase 6 — Garbage Collection

### Q6.1 — Algorithm to find and delete unreachable objects

Mark-and-sweep approach:

Mark phase — build reachable set from refs and HEAD.
Traverse commits → trees → blobs recursively.

Sweep phase — delete objects not in reachable set.

---

### Q6.2 — Race condition between GC and commit

GC may delete objects created during commit before they are referenced.

Git avoids this using:

* Grace period (default ~2 weeks)
* Temporary refs
* Pack-before-delete strategy

---

## Conclusion

The PES-VCS system implements:

* Object storage
* Tree structure
* Index (staging area)
* Commit system
* Version control workflow

---
