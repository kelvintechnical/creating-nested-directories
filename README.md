# Lab: Creating Nested Directories — `mkdir -p`

**Series:** linux-ops-mastery — RHCSA Essential Tools & File Operations
**Subjects covered:** `mkdir` for single-level creation, `-p` for create-all-missing-parents, `-m` for explicit mode on creation, `-v` for verbose output, `-Z` for SELinux context on creation, brace expansion `{a,b,c}` for batch creation, the umask interaction with `-m`, the idempotence of `mkdir -p` (rerunning is harmless), and the project-tree, multi-environment fixture-building patterns used in everyday Linux operations
**Career arcs covered:** RHCSA ("create the directory `/data/X/Y/Z`" exam tasks), RHCE (Ansible `file: state=directory  recurse: yes`), SRE (deploy-time directory provisioning), DevOps (build artifact directory creation), AI/MLOps (experiment-run directory layout — `runs/2026/05/26/exp-1/`)
**Prerequisite:** Labs 05–11 (navigation, listing, copy, move, delete)
**Time Estimate:** 20 to 30 minutes
**Difficulty arc:** Task 1 foundation (`mkdir`) · 2 `-p` recursive parents · 3 `-m` explicit mode · 4 brace expansion fan-out · 5 `-v` and `-Z` modifiers · 6 RHCSA exam-realistic capstone

---

## Objective

Build directory trees in a single command — including missing parents, including specific permissions, including SELinux contexts where needed. By the end of this lab you can answer any "create the path `/data/year/month/day/exp-N/`" prompt with a single `mkdir -p` and you understand why that command is **idempotent** (safe to rerun) while bare `mkdir` is not.

The capstone is an exam-realistic prompt: *"Create the directory tree `/data/projects/{web,api,db}/{logs,configs,backups}` in one command, ensure each leaf directory is mode 0750 owned by `root:wheel`, and verify the SELinux context of one leaf matches `default_t` (or whatever your policy sets)."*

> **Lab safety note:** All work happens in `/tmp/mk-lab` and `/data` (created fresh). No system file is modified.

---

## Concept: `mkdir` Makes Directories — One at a Time, Unless

The bare `mkdir` syscall creates exactly **one** directory and refuses if the parent does not exist. That's by design — the kernel will not invent paths you did not request. Real-world workflows need "make this whole chain" semantics, which is what `-p` adds.

```
   $ mkdir /a/b/c
   mkdir: cannot create directory '/a/b/c': No such file or directory
                 ↑
                 └─ parent '/a/b' doesn't exist; kernel refuses

   $ mkdir -p /a/b/c
   (success — creates /a, then /a/b, then /a/b/c, all in one command)
```

> **Why this matters:** Every script that ensures a directory layout uses `mkdir -p`. It is **idempotent** — running it again is a no-op (no error). Plain `mkdir` errors on the second run, which breaks scripts in CI loops or on Ansible re-application.

---

## 📜 Why `mkdir -p` Exists — The Story

In **Unix v1 (1971)** `mkdir` was a SUID helper because the `mkdir(2)` syscall did not yet exist — you had to construct directory entries by hand. The command's contract was "create one directory; error if the parent is missing." That contract is the same in 2026.

The `-p` flag was added when scripts started needing deep trees. Building tarballs, installing packages, and bootstrapping new systems all required a way to say "ensure this chain exists; if any segment already exists, that's fine; if any segment is missing, create it." Before `-p`, that took a tiny shell loop:

```bash
# pre-`-p` style — verbose, error-prone
for p in /a /a/b /a/b/c; do
  [ -d "$p" ] || mkdir "$p"
done
```

`-p` collapses the loop into one flag. It also turns `mkdir` into one of the few **idempotent** filesystem operations — rerunning it never errors, which is exactly what you want in Ansible, in `Makefile` rules, and in every "ensure this exists" startup sequence.

> **The point of the story:** `mkdir -p` is not "mkdir with creation of parents." It is "mkdir made idempotent." That property is why every modern script uses it instead of bare `mkdir`.

---

## 👪 The mkdir Family — Who Lives There

### `mkdir` flags

| Flag | Meaning |
|---|---|
| (default) | Create one directory; error if parent missing or target exists |
| `-p` / `--parents` | Create missing parents; no error if target exists |
| `-m MODE` / `--mode=MODE` | Set explicit mode (overrides umask) |
| `-v` / `--verbose` | Print each created directory |
| `-Z` (modern coreutils) | Set SELinux context on creation |
| `--context=CONTEXT` | Specific SELinux context |

### Related directory helpers

| Tool | Notes |
|---|---|
| `install -d -m MODE DIR` | Create with mode (and chown via -o/-g) |
| `cp -a SRC DST` | Will create DST and parents as needed |
| `find PATH -type d -mkdir` | (not a real flag — for completeness) |
| Brace expansion `{a,b,c}` | Bash feature, not part of `mkdir`, but creates multiple targets in one command |

> **The point of the family tree:** `mkdir -p` is the canonical "ensure this tree exists" tool. `install -d` is the "ensure this exists with explicit mode/owner" tool, common in Makefiles and rpm specs.

---

## 🔬 The Anatomy of `mkdir -p` — In One Diagram

```
$ mkdir -p /data/projects/web/logs

What the kernel does (one syscall per missing segment):
  1. stat /data            → ENOENT → mkdir /data
  2. stat /data/projects   → ENOENT → mkdir /data/projects
  3. stat /data/projects/web → ENOENT → mkdir /data/projects/web
  4. stat /data/projects/web/logs → ENOENT → mkdir /data/projects/web/logs

What mkdir DOES NOT DO:
  • does not error if any segment already exists (-p makes it idempotent)
  • does not apply -m MODE retroactively to existing parents
  • does not run any hooks or notify other processes
  • does not preserve any special metadata from another path
```

> **Reading rule:** `-p` makes mkdir tolerant of two things — missing intermediate directories, and the final directory already existing. Either of those would have been an error with plain `mkdir`.

---

## 📚 mkdir Reference Table

| Task | Command | Notes |
|---|---|---|
| Create one directory | `mkdir DIR` | Errors if parent missing |
| Create chain | `mkdir -p A/B/C` | Idempotent |
| Create with explicit mode | `mkdir -m 0750 DIR` | Bypasses umask |
| Chain with explicit mode | `mkdir -p -m 0750 A/B/C` | Mode applies to ALL created directories |
| Verbose | `mkdir -pv A/B/C` | Print each created |
| With SELinux context | `mkdir -Z DIR` | Inherits per-policy default |
| Specific context | `mkdir --context='system_u:object_r:tmp_t:s0' DIR` | Less common; usually let `restorecon` fix later |
| Brace expansion fan-out | `mkdir -p /data/{web,api,db}/{logs,configs}` | Creates 3×2 = 6 leaves |
| With install (Makefile-friendly) | `install -d -m 0755 -o root -g wheel /opt/app` | Mode + owner + group in one |
| Followed by chown | `mkdir -p DIR && chown -R user:group DIR` | Separate ownership step |

> **Rule one of `mkdir`:** Always use `-p` in scripts. The idempotence is free and saves you from re-run errors.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | "Create the directory tree X/Y/Z with mode N" tasks are common. `mkdir -p -m` solves them. |
| **RHCE candidate** | Ansible `file: state=directory  recurse: yes  mode: '0750'` mirrors `mkdir -p -m`. |
| **SRE / Platform** | Bootstrap scripts for new hosts run `mkdir -p` everywhere — `/var/log/myapp`, `/opt/myapp`, `/srv/data`. |
| **DevOps** | Dockerfile `RUN mkdir -p /app/{cache,uploads,logs}` is the per-build tree provisioning. |
| **AI / MLOps** | `mkdir -p runs/$(date +%Y)/$(date +%m)/$(date +%d)/exp-$N` standardizes the experiment-output layout. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **plan → mkdir -p → verify** habit.

---

### Task 1 — Create one directory at a time

**Purpose:** Set up the sandbox and use bare `mkdir` to create directories, observing the "parent must exist" rule.

```bash
mkdir -p /tmp/mk-lab && cd /tmp/mk-lab

mkdir single
ls -ld single

mkdir alpha bravo charlie
ls -ld alpha bravo charlie

mkdir deep/nested/path 2>&1 | head -n 1
ls -ld deep 2>&1 | head -n 1
```

**Human-Readable Breakdown:** Make the sandbox, create one directory, then three directories in a single command, then try to create a deep path without `-p` and watch it fail.

**Reading it left to right:** `mkdir DIR` succeeds when the parent exists and the target does not. Multiple-name `mkdir` is just multiple syscalls in one command. Without `-p`, the kernel returns `ENOENT` because the parent is missing.

**The story:** This is the experiment that motivates `-p`. Once you have seen "No such file or directory" on a missing parent, you reach for `-p` every time.

**Expected output:**

```text
drwxr-xr-x. 2 user user 6 May 26 14:50 single
drwxr-xr-x. 2 user user 6 May 26 14:50 alpha
drwxr-xr-x. 2 user user 6 May 26 14:50 bravo
drwxr-xr-x. 2 user user 6 May 26 14:50 charlie
mkdir: cannot create directory 'deep/nested/path': No such file or directory
ls: cannot access 'deep': No such file or directory
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir DIR` | Create one directory |
| `mkdir A B C` | Multiple siblings |
| `ls -ld DIR` | List directory itself, not contents |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `File exists` | Directory already exists — use `-p` to ignore |
| `Permission denied` | No write on parent — `sudo` or pick another path |
| `No such file or directory` | Missing parent — use `-p` |

---

### Task 2 — `mkdir -p` for deep trees

**Purpose:** Use `-p` to create any chain of missing parents in one command, and confirm idempotence by re-running.

```bash
cd /tmp/mk-lab

mkdir -p deep/nested/path
ls -lR deep

# Idempotent — re-run is a no-op
mkdir -p deep/nested/path
ls -lR deep
echo "exit=$?"

# Mixed — some segments exist, some don't
mkdir -p deep/nested/another/branch
ls -lR deep
```

**Human-Readable Breakdown:** Create a three-level deep path in one command. Run the same command again — no error, no change. Then add a new branch to the existing tree.

**Reading it left to right:** `-p` makes `mkdir` walk each segment of the path, creating it if missing and skipping if present. The exit code is `0` whether all segments existed before or not.

**The story:** Once you see `mkdir -p` succeed twice in a row without error, you understand why every Ansible playbook and every CI script uses it. Idempotence is a superpower.

**Expected output:**

```text
deep:
total 0
drwxr-xr-x. 3 user user 21 May 26 14:51 nested

deep/nested:
total 0
drwxr-xr-x. 2 user user 6 May 26 14:51 path

deep/nested/path:
total 0

(same output again)
exit=0

deep:
total 0
drwxr-xr-x. 3 user user 39 May 26 14:51 nested

deep/nested:
total 0
drwxr-xr-x. 2 user user 6 May 26 14:51 another
drwxr-xr-x. 2 user user 6 May 26 14:51 path

deep/nested/another:
total 0
drwxr-xr-x. 2 user user 6 May 26 14:51 branch
```

**Switches**

| Token | Meaning |
|---|---|
| `-p` | Create missing parents; no error if target exists |
| `ls -lR PATH` | Long recursive listing |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Still got `No such file or directory` | Typo in `-p` (looks like `-P`) — both work; check whitespace |
| Permissions wrong on intermediate dirs | `-p` does NOT apply `-m` to existing parents — `chmod` after if needed |
| Exit code non-zero | A real error (permissions, read-only FS) — read the message |

---

### Task 3 — Explicit mode with `-m`

**Purpose:** Use `-m MODE` to set the mode on creation, bypassing umask, and observe how it interacts with `-p`.

```bash
cd /tmp/mk-lab
umask
echo "default umask above"

mkdir default-dir
ls -ld default-dir

mkdir -m 0700 private-dir
ls -ld private-dir

mkdir -p -m 0750 chain/level1/level2
ls -ld chain chain/level1 chain/level1/level2

# Note: -m only applies to NEWLY created dirs
ls -ld chain
mkdir -p -m 0700 chain/level1/already-there/leaf
ls -ld chain chain/level1 chain/level1/already-there chain/level1/already-there/leaf
```

**Human-Readable Breakdown:** Inspect default umask. Create a directory with default mode (umask-derived), then with explicit `-m 0700`. Use `-p -m 0750` to apply mode to a whole new chain. Finally, demonstrate that `-m` does NOT change existing intermediate directories.

**Reading it left to right:** Default `mkdir` uses `0777 & ~umask` for mode. `-m MODE` overrides that. With `-p -m`, every **newly created** segment gets MODE; existing segments are unchanged.

**The story:** Forgetting `-m` and relying on umask is the source of "the directory is mode 775 when I wanted 750" bugs. Always set `-m` explicitly when the prompt cares about permissions.

**Expected output:**

```text
0022
default umask above
drwxr-xr-x. 2 user user 6 May 26 14:53 default-dir
drwx------. 2 user user 6 May 26 14:53 private-dir
drwxr-x---. 3 user user 21 May 26 14:53 chain
drwxr-x---. 3 user user 21 May 26 14:53 chain/level1
drwxr-x---. 2 user user 6 May 26 14:53 chain/level1/level2
drwxr-x---. 3 user user 21 May 26 14:53 chain
drwx------. 3 user user 21 May 26 14:53 chain/level1/already-there
drwx------. 2 user user 6 May 26 14:53 chain/level1/already-there/leaf
```

**Switches**

| Token | Meaning |
|---|---|
| `-m 0750` | Set mode at creation (0750 = rwxr-x---) |
| `umask` | Show or set the default mode mask |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Mode is `0775` instead of `0750` | Forgot `-m`; umask was 0022 |
| Existing parent has wrong mode | `-m` does not retroactively chmod — use `chmod` separately |
| Permission denied creating a leaf | Intermediate dir mode does not allow execute — re-set with chmod |

---

### Task 4 — Brace expansion fan-out

**Purpose:** Use bash brace expansion `{a,b,c}` to create many sibling directories in one command, and combine with `-p` for multi-level fan-outs.

```bash
cd /tmp/mk-lab

mkdir -p projects/{web,api,db}
ls projects/

mkdir -p projects/{web,api,db}/{logs,configs,backups}
find projects -type d

# Year/month/day pattern
mkdir -p runs/2026/{01..12}/{01..31}
find runs -type d | head -n 20

# Cartesian product
mkdir -p envs/{dev,stage,prod}/{us-east,us-west,eu-west}
find envs -type d
```

**Human-Readable Breakdown:** Fan out three siblings under `projects/`, then three sub-siblings inside each (9 leaves), then a year/month/day fixture (12*31 = 372 leaves under runs/2026), then a 3×3 environment matrix.

**Reading it left to right:** `{a,b,c}` is bash brace expansion — the shell expands it into all combinations **before** `mkdir` runs. `{01..12}` is a sequence (compact form). Combining expansions multiplies — `{web,api,db}/{logs,configs}` produces 6 paths.

**The story:** Brace expansion turns long shell loops into one-liners. Once you have set up a year/month/day directory tree with one command, you stop writing for-loops for fixture creation.

**Expected output:**

```text
api  db  web
projects
projects/api
projects/api/backups
projects/api/configs
projects/api/logs
projects/db
projects/db/backups
projects/db/configs
projects/db/logs
projects/web
projects/web/backups
projects/web/configs
projects/web/logs
runs
runs/2026
runs/2026/01
runs/2026/01/01
runs/2026/01/02
...
envs
envs/dev
envs/dev/eu-west
envs/dev/us-east
envs/dev/us-west
envs/prod
...
```

**Switches**

| Token | Meaning |
|---|---|
| `{a,b,c}` | Brace expansion — comma-separated list |
| `{01..12}` | Numeric sequence (compact form) |
| `find DIR -type d` | List all directories recursively |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Brace expansion not happening | Used quotes — `'{a,b,c}'` is literal; remove quotes around braces |
| Path created literally with braces | Running `sh` instead of `bash` — switch shells |
| Too many directories created | Multiply: N × M segments; double-check product before running |

---

### Task 5 — Verbose `-v` and SELinux `-Z`

**Purpose:** Use `-v` for an audit trail of created directories, and `-Z` to set SELinux context at creation (or rely on policy inheritance).

```bash
cd /tmp/mk-lab

mkdir -pv audit/{2026/{Q1,Q2,Q3,Q4},archive}
ls -lZd audit audit/2026 audit/2026/Q1

# -Z with no argument inherits the per-policy default
mkdir -Z secure-tmp
ls -ldZ secure-tmp

# Explicit context (rare; usually wrong unless policy says so)
mkdir --context='unconfined_u:object_r:user_tmp_t:s0' explicit-ctx
ls -ldZ explicit-ctx
```

**Human-Readable Breakdown:** Create an audit tree with `-pv` for a verbose log. Then create directories with `-Z` (policy default) and `--context=` (explicit).

**Reading it left to right:** `-v` prints `mkdir: created directory '...'` for each segment. `-Z` (with no argument) lets the kernel apply policy-derived contexts. Explicit `--context=` overrides — useful for fixing wrong defaults during fixture creation.

**The story:** Logging is invaluable in scripts that bootstrap a system from scratch. `-v` makes that audit trail free.

**Expected output:**

```text
mkdir: created directory 'audit'
mkdir: created directory 'audit/2026'
mkdir: created directory 'audit/2026/Q1'
mkdir: created directory 'audit/2026/Q2'
mkdir: created directory 'audit/2026/Q3'
mkdir: created directory 'audit/2026/Q4'
mkdir: created directory 'audit/archive'
drwxr-xr-x. 4 user user unconfined_u:object_r:user_tmp_t:s0 30 May 26 14:55 audit
drwxr-xr-x. 6 user user unconfined_u:object_r:user_tmp_t:s0 32 May 26 14:55 audit/2026
drwxr-xr-x. 2 user user unconfined_u:object_r:user_tmp_t:s0  6 May 26 14:55 audit/2026/Q1
drwxr-xr-x. 2 user user unconfined_u:object_r:user_tmp_t:s0  6 May 26 14:55 secure-tmp
drwxr-xr-x. 2 user user unconfined_u:object_r:user_tmp_t:s0  6 May 26 14:55 explicit-ctx
```

**Switches**

| Token | Meaning |
|---|---|
| `-v` | Verbose — print each creation |
| `-Z` | SELinux context (policy default) |
| `--context=CTX` | Explicit context |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-Z` not recognized | Old coreutils — upgrade or use `chcon` after |
| Wrong context inherited | Use `restorecon -v PATH` after to apply policy defaults |
| Explicit context error | Verify with `semanage fcontext -l` what is allowed |

---

### Task 6 — Capstone: RHCSA-realistic deep-tree provisioning

**Task statement:** *"Create the directory tree `/data/projects/{web,api,db}/{logs,configs,backups}` in one command. Each leaf directory must be mode `0750` owned by `root:wheel`. Save a recursive listing to `/root/projects-tree.txt` and verify by checking the mode of one leaf."*

**Purpose:** Execute a deep-tree provisioning end-to-end with correct mode and ownership.

```bash
sudo -i
groupadd -f wheel 2>/dev/null || true

rm -rf /data/projects
mkdir -p -m 0750 /data/projects/{web,api,db}/{logs,configs,backups}

chown -R root:wheel /data/projects

ls -lR /data/projects > /root/projects-tree.txt

ls -ld /data/projects/web/logs
stat -c '%n mode=%a owner=%U group=%G' /data/projects/web/logs

# Verification
test -d /data/projects/web/logs && echo "VERIFY: leaf directory exists"
test "$(stat -c '%a' /data/projects/web/logs)" = "750" && echo "VERIFY: mode is 0750"
test "$(stat -c '%U' /data/projects/web/logs)" = "root" && echo "VERIFY: owner is root"
test "$(stat -c '%G' /data/projects/web/logs)" = "wheel" && echo "VERIFY: group is wheel"
wc -l /root/projects-tree.txt
```

**Human-Readable Breakdown:** Become root, ensure the `wheel` group exists, build a clean `/data/projects` tree with `mkdir -p -m 0750` (9 leaves), recursively chown to `root:wheel`, write the recursive listing to `/root/projects-tree.txt`, and run four `test`-based checks for mode/owner/group.

**Layer stack you built:**

```text
/data/projects/                    mode=0750 root:wheel
   ├── web/
   │     ├── logs/
   │     ├── configs/
   │     └── backups/
   ├── api/
   │     ├── logs/
   │     ├── configs/
   │     └── backups/
   └── db/
         ├── logs/
         ├── configs/
         └── backups/
```

**The story:** This is the **canonical 30-second exam answer** for deep-tree provisioning. Memorize the spine: `mkdir -p -m MODE PARENT/{A,B,C}/{X,Y,Z}` → `chown -R USER:GROUP PARENT` → verify with `stat`.

**Expected verification output:**

```text
drwxr-x---. 2 root wheel 6 May 26 14:57 /data/projects/web/logs
/data/projects/web/logs mode=750 owner=root group=wheel
VERIFY: leaf directory exists
VERIFY: mode is 0750
VERIFY: owner is root
VERIFY: group is wheel
26 /root/projects-tree.txt
```

**Cleanup**

```bash
rm -rf /tmp/mk-lab /data/projects /root/projects-tree.txt
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Leaves got mode 0700 instead of 0750 | umask was 0077 — explicit `-m 0750` overrides |
| `wheel` group missing | `groupadd wheel` (RHEL has it by default) |
| `chown -R` skipped some files | `-R` only affects files inside; not the args themselves — that's fine for dirs |
| `Permission denied` writing report | Not root — `sudo -i` |

---

## 🔍 mkdir Decision Guide

```
Need to create a directory?
  │
  ├── "One directory, parent exists"
  │       └── ✅ mkdir DIR
  │
  ├── "Multi-level tree, parents may be missing"
  │       └── ✅ mkdir -p A/B/C
  │
  ├── "Same as above, with explicit mode for ALL new segments"
  │       └── ✅ mkdir -p -m 0750 A/B/C
  │
  ├── "Several siblings at once"
  │       └── ✅ mkdir -p PARENT/{a,b,c}
  │
  ├── "Cartesian product of siblings"
  │       └── ✅ mkdir -p PARENT/{a,b,c}/{x,y,z}
  │
  ├── "Year/month/day fixture"
  │       └── ✅ mkdir -p runs/2026/{01..12}/{01..31}
  │
  ├── "Need a per-policy SELinux context"
  │       └── ✅ mkdir -Z DIR        (or use `restorecon` after)
  │
  ├── "Need ownership too — Makefile-style"
  │       └── ✅ install -d -m 0755 -o root -g wheel /opt/app
  │
  └── "Idempotent script line"
          └── ✅ mkdir -p DIR        (always; never plain mkdir)
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Set up `/tmp/mk-lab` and create directories with plain `mkdir`
- [ ] 02 Build a three-level tree with `mkdir -p` and verify idempotence
- [ ] 03 Use `-m MODE` to bypass umask on creation
- [ ] 04 Fan out trees with brace expansion `{a,b,c}/{x,y,z}` and `{01..31}`
- [ ] 05 Use `-v` for audit, `-Z` for SELinux context
- [ ] 06 Execute the RHCSA capstone — build `/data/projects/{web,api,db}/{logs,configs,backups}` mode 0750 owned root:wheel

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Forgot `-p` in script | Errors on re-run | Always `-p` in scripts |
| `-m` did not affect existing parent | mode unchanged on `/data` | `-m` applies only to newly created segments — use `chmod` for existing |
| Brace expansion quoted | Literal `{a,b}` directory created | Remove quotes around braces |
| Wanted parents with one mode and leaf with another | All segments get the same mode | Two `mkdir -p -m ...` calls |
| Used `mkdir` instead of `install -d` in Makefile | No way to set owner | Use `install -d -m -o -g` |
| Created tree without setting context | Files inherit wrong SELinux type | `restorecon -RFv /target/path` after |
| Wrong umask | Mode is permissive (0775 instead of 0750) | Always `-m` when mode matters |
| Tree under bind mount | Lost after umount | Verify mount points before |
| Tried to `mkdir` over an existing file | `cannot create directory ...: File exists` | Remove the file or pick a different name |
| Mass-created millions of dirs | Filesystem inode exhaustion possible | Check `df -i` before |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- One-liner reflex: `mkdir -p -m MODE /path/{A,B,C}/{X,Y,Z}` then `chown -R user:group /path`. Practice until automatic.

**RHCE candidate**
- Ansible: `file: state=directory  recurse: yes  mode: '0750'  owner: root  group: wheel  path: /data/projects/web/logs`.

**SRE / Platform interview**
- "How would you ensure a deep directory tree exists at host bootstrap?" → `mkdir -p` plus `chown -R` plus `restorecon -RFv` if SELinux context matters.

**DevOps**
- Dockerfiles use `RUN mkdir -p` to provision per-build directories; rebuilds reuse the same layer.

**AI / MLOps**
- `mkdir -p runs/$(date +%Y/%m/%d)/exp-$RUN_ID` standardizes output paths across experiments.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Directory Navigation | You navigate the trees `mkdir -p` builds |
| Lab 06 — Listing & SELinux Contexts | Verifying contexts after `mkdir -Z` |
| Lab 08 — Copying Files | Often paired: `mkdir -p DST && cp -a SRC DST/` |
| Lab 11 — Safe Deletion | The undo for created trees |
| Lab — Standard Permissions (`chmod`) *(later)* | The post-creation permission tweak |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
