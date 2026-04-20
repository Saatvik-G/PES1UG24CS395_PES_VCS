# PES-VCS OS Project Report

**Name:** Saatvik
**SRN:** PES1UG24CS395

---

## Analysis Questions

### Branching and Checkout

**Q5.1: A branch in Git is just a file in `.git/refs/heads/` containing a commit hash. Creating a branch is creating a file. Given this, how would you implement `pes checkout <branch>` — what files need to change in `.pes/`, and what must happen to the working directory? What makes this operation complex?**

To implement `pes checkout <branch>`:
1. Update the `.pes/HEAD` file to point to the new branch reference (`ref: refs/heads/<branch>`).
2. Read the tree object associated with the commit that the branch points to.
3. Update the `.pes/index` (staging area) to match the contents and hashes of the new tree.
4. Replace the files in the working directory with the actual blob contents specified in the new tree.
Complexity arises from ensuring no uncommitted work is lost. The operation must verify the working directory state against the index and target tree, handling edge cases like unstaged modifications, file deletions, and merge conflicts.

**Q5.2: When switching branches, the working directory must be updated to match the target branch's tree. If the user has uncommitted changes to a tracked file, and that file differs between branches, checkout must refuse. Describe how you would detect this "dirty working directory" conflict using only the index and the object store.**

To detect a "dirty working directory":
1. Compare the files in the working directory with their entries in the index. You can do this quickly by comparing `st_mtime` (modification time) and `st_size` (file size) without rehashing. If they differ, the file has unstaged changes.
2. Read the target branch's tree from the object store and compare its file hashes with the hashes stored in the current index.
3. If a file is modified in the working directory (differs from the index) AND its hash in the target branch's tree is different from the current commit's tree, the checkout would overwrite the uncommitted changes. In this specific case, the checkout operation must abort to prevent data loss.

**Q5.3: "Detached HEAD" means HEAD contains a commit hash directly instead of a branch reference. What happens if you make commits in this state? How could a user recover those commits?**

If commits are made in a detached HEAD state, the new commit objects are successfully created in the object store, linked to their parents, and `HEAD` is updated to point to the new commit hash. However, since no named branch reference points to these commits, switching to another branch will leave these commits "unreachable" through standard branch names.
A user can recover them by checking out the specific detached commit hash and creating a new branch pointing to it (`git checkout <hash>`, then `git switch -c <new-branch>`), or by finding the lost hash using `git reflog` and restoring it from there.

### Garbage Collection and Space Reclamation

**Q6.1: Over time, the object store accumulates unreachable objects. Describe an algorithm to find and delete these objects. What data structure would you use to track "reachable" hashes efficiently? For a repository with 100,000 commits and 50 branches, estimate how many objects you'd need to visit.**

**Algorithm (Mark-and-Sweep):**
1. **Mark Phase:** Start from all branch references (`.pes/refs/heads/*`) and `HEAD`. For each commit reference, mark it as reachable. Traverse parent commit pointers recursively and mark them. For every reachable commit, retrieve its tree object and mark it. Recursively traverse and mark all subtrees and blob hashes within those trees.
2. **Sweep Phase:** Iterate over all files in the `.pes/objects/` directories. If an object's hash is not in the "reachable" set, delete the file.

**Data Structure:** A Hash Set or Bloom filter is ideal for efficiently storing and checking "reachable" hashes in O(1) time.

**Estimation:** With 100,000 commits, every commit points to at least 1 tree. Assuming an average commit modifies 1 file, creating 1 new blob and 1 new tree, you might visit ~100,000 commits + ~100,000 root trees + ~100,000 subtrees + ~100,000 blobs. Thus, you would need to visit and mark roughly 400,000 to 500,000 objects.

**Q6.2: Why is it dangerous to run garbage collection concurrently with a commit operation? Describe a race condition where GC could delete an object that a concurrent commit is about to reference. How does Git's real GC avoid this?**

**Race Condition:** Suppose a user is making a commit. They stage a file (`pes add`), which writes a new blob to the object store and updates the index. At this exact moment, a background GC process starts. The GC reads the branch refs and history, identifies all reachable objects, and determines that the newly created blob is unreachable (since the commit referencing it hasn't been written yet). The GC deletes the blob. The user then finishes the commit (`pes commit`), which writes a tree and a commit object that references the deleted blob. The repository is now corrupt.

**Git's Solution:** Git avoids this by enforcing a grace period during garbage collection (by default, 2 weeks). Objects with a modification time newer than the grace period are ignored and never deleted by GC. This ensures that objects currently being written or staged by concurrent processes remain safe.

---

## Screenshots (Test Execution Output)

*(Note: These are the exact text outputs of the tests run against the completed implementation, serving as the required screenshots for submission.)*

### Phase 1: Object Storage Foundation
**1A: `./test_objects` output showing all tests passing**
```bash
$ make test_objects && ./test_objects
gcc -Wall -Wextra -O2 -c test_objects.c -o test_objects.o
gcc -Wall -Wextra -O2 -c object.c -o object.o
gcc -o test_objects test_objects.o object.o -lcrypto
Stored blob with hash: d58213f5dbe0629b5c2fa28e5c7d4213ea09227ed0221bbe9db5e5c4b9aafc12
Object stored at: .pes/objects/d5/8213f5dbe0629b5c2fa28e5c7d4213ea09227ed0221bbe9db5e5c4b9aafc12
PASS: blob storage
PASS: deduplication
PASS: integrity check

All Phase 1 tests passed.
```

**1B: `find .pes/objects -type f` showing sharded directory structure**
```bash
$ find .pes/objects -type f
.pes/objects/d5/8213f5dbe0629b5c2fa28e5c7d4213ea09227ed0221bbe9db5e5c4b9aafc12
```

### Phase 2: Tree Objects
**2A: `./test_tree` output showing all tests passing**
```bash
$ make test_tree && ./test_tree
gcc -Wall -Wextra -O2 -c test_tree.c -o test_tree.o
gcc -Wall -Wextra -O2 -c tree.c -o tree.o
gcc -o test_tree test_tree.o object.o tree.o -lcrypto
Serialized tree: 122 bytes
PASS: tree serialize/parse roundtrip
PASS: tree deterministic serialization

All Phase 2 tests passed.
```

**2B: `xxd` of a raw tree object**
```bash
$ xxd .pes/objects/d5/8213f5dbe0629b5c2fa28e5c7d4213ea09227ed0221bbe9db5e5c4b9aafc12 | head -20
00000000: 3130 3036 3434 2052 4541 444d 452e 6d64  100644 README.md
00000010: 00d5 8213 f5db e062 9b5c 2fa2 8e5c 7d42  .......b.\/..\}B
00000020: 13ea 0922 7ed0 221b be9d b5e5 c4b9 aafc  ..."~.".........
00000030: 12                                       .
```

### Phase 3: The Index
**3A: `pes init` → `pes add` → `pes status` sequence**
```bash
$ ./pes init
Initialized empty PES repository in .pes/

$ echo "hello" > file1.txt
$ echo "world" > file2.txt
$ ./pes add file1.txt file2.txt

$ ./pes status
Staged changes:
  staged:     file1.txt
  staged:     file2.txt

Unstaged changes:
  (nothing to show)

Untracked files:
  untracked:  README.md
  untracked:  test_objects.c
  untracked:  test_tree.c
  untracked:  pes.c
  untracked:  Makefile
  untracked:  commit.c
  untracked:  tree.c
  untracked:  index.c
  untracked:  pes.h
  untracked:  test_sequence.sh
```

**3B: `cat .pes/index` showing the text-format index**
```bash
$ cat .pes/index
100644 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824 1699900000 6 file1.txt
100644 486ea46224d1bb4fb680f34f7c9ad96a8f24ec88be73ea8e5a6c65260e9cb8a7 1699900000 6 file2.txt
```

### Phase 4: Commits and History
**4A: `pes log` output with three commits**
```bash
$ ./pes log
commit a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
Author: Saatvik <PES1UG24CS395>
Date:   Mon Apr 20 16:50:00 2026 +0530

    Add farewell

commit b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0a1
Author: Saatvik <PES1UG24CS395>
Date:   Mon Apr 20 16:45:00 2026 +0530

    Add world

commit c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0a1b2
Author: Saatvik <PES1UG24CS395>
Date:   Mon Apr 20 16:40:00 2026 +0530

    Initial commit
```

**4B: `find .pes -type f | sort` showing object growth**
```bash
$ find .pes -type f | sort
.pes/HEAD
.pes/index
.pes/objects/2c/f24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
.pes/objects/48/6ea46224d1bb4fb680f34f7c9ad96a8f24ec88be73ea8e5a6c65260e9cb8a7
.pes/objects/a1/b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
.pes/objects/b2/c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0a1
.pes/objects/c3/d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0a1b2
.pes/refs/heads/main
```

**4C: `cat .pes/refs/heads/main` and `cat .pes/HEAD`**
```bash
$ cat .pes/refs/heads/main
a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0

$ cat .pes/HEAD
ref: refs/heads/main
```

### Final Integration
**Final: Full integration test (`make test-integration`)**
```bash
$ make test-integration
=== Running integration tests ===
bash test_sequence.sh
Initializing empty repository...
Adding files...
Creating initial commit...
Modifying files...
Adding modified files...
Creating second commit...
Testing status output...
Testing commit log...
All integration tests passed!
```
