# Lab 12: Creating Nested Directories — `mkdir -p`

**Series:** File Operations & Shell Fundamentals · **Lab 12 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (foundational), RHCE EX294 (every Ansible path reference), CKA (every kubelet/etcd/containerd path), RHCA building blocks (RH342 troubleshooting, RH358 services, RH236 storage)  
**Prerequisite:** Labs 05–11  
**Time Estimate:** 30–40 minutes  
**Difficulty arc:** Tasks 1–6 foundation · 7–12 practical · 13–17 advanced · 18–20 exam-realistic

---

## 🎯 Objective

Build directory trees of any depth in one safe, idempotent command. By the end you will use `mkdir -p` so naturally that you never see "No such file or directory" again when prepping a path for a service, Ansible role, or kubelet manifest.

---

## 🧠 Concept: Directories Are Just Special Files

In Linux, a directory is a file whose contents are a list of `name → inode` pairs. To create one, the kernel:

1. Allocates an inode (with type `d`).
2. Initializes its content with two entries: `.` (self) and `..` (parent).
3. Adds a new entry in the parent directory pointing to the new inode.

```
mkdir /a/b/c
   └── needs /a/b to already exist
        └── if /a/b doesn't exist → "No such file or directory"

mkdir -p /a/b/c
   └── walks the path; creates each missing piece
        └── never errors if /a, /a/b, or /a/b/c already exist
```

> **Why this matters on every cert:** RHCSA Task 8, CKA "rebuild `/etc/kubernetes/pki/etcd`," Ansible role scaffolding — every one of them creates nested paths. `mkdir -p` is the only safe way.

### The `mkdir -p` invariants

| Invariant | Meaning |
|---|---|
| **Idempotent** | Running twice produces the same result; second run is a no-op |
| **Recursive** | Creates parents in one call |
| **No error on existing** | "Already there" is success, not failure |
| **Inherits umask** | Final permissions = `0777 & ~umask` (umask `0022` → `0755`) |
| **Each intermediate dir** | Gets the same default permissions (`-m` only applies to the final or all if `-m -p`) |

---

## 📚 `mkdir` Reference

| Flag | Long form | Purpose |
|---|---|---|
| (none) | — | Create a single directory; error if parent missing or it already exists |
| `-p` | `--parents` | Create missing parents; do not error if any segment exists |
| `-m MODE` | `--mode=MODE` | Set permission mode at creation (bypasses umask) |
| `-v` | `--verbose` | Print each directory created |
| `-Z` | `--context=CTX` | Set SELinux context at creation (or use system default if no CTX) |
| `--context=CTX` | — | Explicit SELinux context |

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **RHCSA EX200** | Task 8 (create directory with specific mode/owner), every "create the path X" prerequisite |
| **RHCE EX294** | `ansible.builtin.file: state: directory` mirrors `mkdir -p` semantics |
| **CKA** | Rebuilding `/etc/kubernetes/pki/etcd`, `/var/lib/kubelet/pods`, `/etc/cni/net.d` |
| **RHCA — RH342 (Troubleshooting)** | Recovery — recreate service directories with correct mode/owner/context |
| **RHCA — RH358 (Services)** | Each service has a tree under `/etc/<svc>/conf.d` and `/var/lib/<svc>` |
| **RHCA — RH236 (Gluster)** | Brick paths under `/data/gluster/brickN/` need `mkdir -p` + relabel |

---

## 🔧 The 20 Tasks

> Each task ends with three callouts: **Switches** (every flag), **Output decoded** (what each line means), and **Troubleshoot** (what to do if it goes wrong).

---

### Task 1 — Set up the lab workspace

**Purpose:** A clean root so each subsequent step starts predictable.

```bash
mkdir -p ~/mkdir-lab
cd ~/mkdir-lab
pwd
```

**Expected output:**

```
/home/ec2-user/mkdir-lab
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p` | Create parents; ignore "already exists" |
| `cd` | Change to the new dir |
| `pwd` | Confirm |

**Output decoded**

| Line | Meaning |
|---|---|
| `/home/ec2-user/mkdir-lab` | Workspace ready |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Permission denied` | Use a path inside your home |

---

### Task 2 — `mkdir` basic — single directory

**Purpose:** The simplest case. Single existing parent, single new child.

```bash
mkdir simple
ls -ld simple
```

**Expected output:**

```
drwxr-xr-x. 2 ec2-user ec2-user 6 Sep 12 16:00 simple
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir DIR` | Create one directory |
| `ls -ld DIR` | Show the directory itself, not its contents |

**Output decoded**

| Column | Value | Meaning |
|---|---|---|
| Type+perms | `drwxr-xr-x.` | Directory, owner full, group r-x, other r-x |
| Links | `2` | Self (`.`) plus name in parent |
| Owner/Group | `ec2-user ec2-user` | Defaults to your identity |
| Size | `6` | Directory metadata size (not contents) |
| Time | `Sep 12 16:00` | Created just now |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Permissions are 0700 instead of 0755 | Your `umask` is `0077` — run `umask 0022` and recreate |

---

### Task 3 — `mkdir` multiple at once

**Purpose:** Pass multiple names to create them in parallel.

```bash
mkdir alpha beta gamma
ls -d alpha beta gamma
```

**Expected output:**

```
alpha  beta  gamma
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir a b c` | Three separate top-level directories |
| `ls -d` | List the directory entries, not their contents |

**Output decoded**

| Token | Meaning |
|---|---|
| All three names | Created in one syscall sequence |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mkdir: cannot create directory ‘beta’: File exists` | One of them already exists — add `-p` or remove first |

---

### Task 4 — Plain `mkdir` fails on nested paths

**Purpose:** Show why `-p` exists. Without it, `mkdir a/b/c` fails if `a/b` is missing.

```bash
mkdir a/b/c 2>&1
```

**Expected output:**

```
mkdir: cannot create directory 'a/b/c': No such file or directory
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir a/b/c` | Tries to create `c` in `a/b`; refuses because `a/b` doesn't exist |
| `2>&1` | Combine stderr so we see the error |

**Output decoded**

| Line | Meaning |
|---|---|
| `No such file or directory` | Parent path is missing |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wanted to create the whole tree | Use `-p` (Task 5) |

---

### Task 5 — `mkdir -p` builds the whole chain

**Purpose:** The headline feature.

```bash
mkdir -p a/b/c/d/e
ls -R a
```

**Expected output:**

```
a:
b

a/b:
c

a/b/c:
d

a/b/c/d:
e

a/b/c/d/e:
```

**Switches**

| Flag | Meaning |
|---|---|
| `-p` | Create missing parents; never error on existing |

**Output decoded**

| Each block | One level of the chain; final `e` is empty |

**Why a sysadmin needs this:** RHCSA Task 8 commonly says "create `/srv/web/sites/example.com/htdocs`" — a single `mkdir -p` handles it.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Strange permissions mid-chain | Use `-m` (Task 9) to set them explicitly |

---

### Task 6 — Idempotent: `mkdir -p` on existing dir is a no-op

**Purpose:** Prove that running `mkdir -p` twice never errors. This is what makes it safe for scripts.

```bash
mkdir -p ~/mkdir-lab/already
mkdir -p ~/mkdir-lab/already
echo "exit code = $?"
```

**Expected output:**

```
exit code = 0
```

**Switches**

| Token | Meaning |
|---|---|
| `-p` | Suppress "already exists" errors |

**Output decoded**

| Line | Meaning |
|---|---|
| `exit code = 0` | Success even on the second run |

**Why on RHCE EX294:** Ansible's `file: state=directory` is idempotent for the same reason — internally it acts like `mkdir -p`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Exit code 1 on second run | You forgot `-p` — re-run with it |

---

### Task 7 — Brace expansion to make many at once

**Purpose:** Combine `mkdir -p` with brace expansion for complex trees in one line.

```bash
mkdir -p ~/mkdir-lab/web/{logs,configs,htdocs/{en,es,fr},backup/{daily,weekly}}
find ~/mkdir-lab/web -type d | sort
```

**Expected output:**

```
/home/ec2-user/mkdir-lab/web
/home/ec2-user/mkdir-lab/web/backup
/home/ec2-user/mkdir-lab/web/backup/daily
/home/ec2-user/mkdir-lab/web/backup/weekly
/home/ec2-user/mkdir-lab/web/configs
/home/ec2-user/mkdir-lab/web/htdocs
/home/ec2-user/mkdir-lab/web/htdocs/en
/home/ec2-user/mkdir-lab/web/htdocs/es
/home/ec2-user/mkdir-lab/web/htdocs/fr
/home/ec2-user/mkdir-lab/web/logs
```

**Switches**

| Token | Meaning |
|---|---|
| `{a,b,c}` | Comma brace expansion |
| `{a/{x,y},b}` | Nested brace expansion (cartesian product) |
| `find ... -type d` | List directories only |

**Output decoded**

| Each line | A subdirectory the shell expanded and `mkdir -p` created |

**Why a sysadmin needs this on RHCE EX294:** Ansible role scaffolding — `roles/myapp/{tasks,handlers,templates,files,vars,defaults,meta}` in one command.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Shell made one literal directory named `{a,b,c}` | You're in `sh` not `bash` — switch shells |

---

### Task 8 — Verbose with `-v`

**Purpose:** See what `mkdir -p` actually created (useful when chains are deep).

```bash
mkdir -pv ~/mkdir-lab/v/i/s/i/b/l/e
```

**Expected output:**

```
mkdir: created directory '/home/ec2-user/mkdir-lab/v'
mkdir: created directory '/home/ec2-user/mkdir-lab/v/i'
mkdir: created directory '/home/ec2-user/mkdir-lab/v/i/s'
mkdir: created directory '/home/ec2-user/mkdir-lab/v/i/s/i'
mkdir: created directory '/home/ec2-user/mkdir-lab/v/i/s/i/b'
mkdir: created directory '/home/ec2-user/mkdir-lab/v/i/s/i/b/l'
mkdir: created directory '/home/ec2-user/mkdir-lab/v/i/s/i/b/l/e'
```

**Switches**

| Flag | Meaning |
|---|---|
| `-v` | Verbose — print one line per directory actually created |

**Output decoded**

| Line | Meaning |
|---|---|
| `created directory '...'` | A new directory at that depth |
| (no line for existing dirs) | `-v` only mentions NEW creations |

**Why on RHCA RH342:** During recovery, `-v` proves which segments you actually fixed.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No output at all | Every segment already existed — that's success, not failure |

---

### Task 9 — Set mode at creation with `-m`

**Purpose:** Avoid a separate `chmod` by setting mode at creation.

```bash
mkdir -m 0750 ~/mkdir-lab/secret
ls -ld ~/mkdir-lab/secret
mkdir -m 0700 -p ~/mkdir-lab/keys/private/v1
ls -ld ~/mkdir-lab/keys ~/mkdir-lab/keys/private ~/mkdir-lab/keys/private/v1
```

**Expected output:**

```
drwxr-x---. 2 ec2-user ec2-user 6 Sep 12 16:10 /home/ec2-user/mkdir-lab/secret
drwxr-xr-x. 3 ec2-user ec2-user 21 Sep 12 16:10 /home/ec2-user/mkdir-lab/keys
drwxr-xr-x. 3 ec2-user ec2-user 17 Sep 12 16:10 /home/ec2-user/mkdir-lab/keys/private
drwx------. 2 ec2-user ec2-user  6 Sep 12 16:10 /home/ec2-user/mkdir-lab/keys/private/v1
```

**Switches**

| Flag | Meaning |
|---|---|
| `-m MODE` | Set permission mode at creation |
| `0750` | Owner rwx, group r-x, other none |
| `0700` | Owner rwx only |

**Output decoded**

| Token | Meaning |
|---|---|
| `drwxr-x---.` | Mode 0750 — task explicit |
| Intermediate `keys` and `keys/private` showing `0755` | `-m` applies only to the **final** directory created |
| Final `v1` at `drwx------.` (0700) | Got the requested mode |

> **Gotcha:** With `-p`, intermediate directories use the default mode (`0777 & ~umask`), NOT the `-m` mode. If you need every level at 0700, use a loop:
> ```bash
> for d in keys keys/private keys/private/v1; do mkdir -m 0700 -p "$d"; done
> ```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wrong mode on intermediates | Set them manually with `chmod -R 0700 keys` or use the loop pattern above |

---

### Task 10 — Set SELinux context with `-Z`

**Purpose:** Create with the policy-defined context — no separate `restorecon` needed.

```bash
sudo mkdir -p -Z /var/www/html/site2
sudo ls -lZd /var/www/html /var/www/html/site2
```

**Expected output:**

```
drwxr-xr-x. 3 root root system_u:object_r:httpd_sys_content_t:s0 19 Sep 12 16:15 /var/www/html
drwxr-xr-x. 2 root root system_u:object_r:httpd_sys_content_t:s0  6 Sep 12 16:15 /var/www/html/site2
```

**Switches**

| Flag | Meaning |
|---|---|
| `-Z` | Set SELinux context (no value → use the system default for the path) |
| `--context=CTX` | Explicit context |

**Output decoded**

| Token | Meaning |
|---|---|
| `httpd_sys_content_t` | Inherited from parent / policy — what Apache needs |

**Why a sysadmin needs this on RHCSA Task 16:** Avoids the `restorecon` follow-up step — context is correct from creation.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mkdir: unrecognized option -- 'Z'` | Old coreutils — fall back to `mkdir` + `restorecon -Rv` |

---

### Task 11 — Create + chown + chmod pattern

**Purpose:** Most exam tasks require all three. Know the canonical 3-line incantation.

```bash
sudo mkdir -p /srv/admin/reports
sudo chown root:wheel /srv/admin/reports 2>/dev/null || sudo chown root:root /srv/admin/reports
sudo chmod 0750 /srv/admin/reports
ls -ldZ /srv/admin/reports
```

**Expected output:**

```
drwxr-x---. 2 root root unconfined_u:object_r:var_t:s0 6 Sep 12 16:20 /srv/admin/reports
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p` | Create dir tree |
| `chown user:group` | Set owner+group |
| `chmod MODE` | Set permissions |
| `ls -ldZ` | Verify in one line (directory itself + context) |

**Output decoded**

| Token | Meaning |
|---|---|
| `drwxr-x---.` | Mode 0750 |
| `root root` | Owner+group as set |
| `var_t` | Default context for `/srv` — may need `restorecon` or `semanage fcontext` depending on task |

**Why on RHCSA Task 8:** Single most common exam task. Memorize the 4-line pattern (mkdir, chown, chmod, ls -ldZ).

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `chown: invalid group: 'wheel'` | Wheel group may not exist on your distro — substitute `root` or the correct group |

---

### Task 12 — Absolute vs relative paths

**Purpose:** Both work. Pick the right one for the situation.

```bash
cd ~/mkdir-lab
mkdir -pv ./relative/deep/dir
mkdir -pv /tmp/absolute/$USER/deep/dir
ls -d ./relative/deep/dir /tmp/absolute/$USER/deep/dir
```

**Expected output:**

```
mkdir: created directory './relative'
mkdir: created directory './relative/deep'
mkdir: created directory './relative/deep/dir'
mkdir: created directory '/tmp/absolute/ec2-user'
mkdir: created directory '/tmp/absolute/ec2-user/deep'
mkdir: created directory '/tmp/absolute/ec2-user/deep/dir'
./relative/deep/dir  /tmp/absolute/ec2-user/deep/dir
```

**Switches**

| Token | Meaning |
|---|---|
| `./relative/...` | Relative — depends on current dir |
| `/tmp/absolute/...` | Absolute — works from anywhere |
| `$USER` | Environment variable for your username |

**Output decoded**

| Phase | Where the directory landed |
|---|---|
| Relative | Under `~/mkdir-lab/` (because that's `pwd`) |
| Absolute | Under `/tmp/` (regardless of `pwd`) |

**Why a sysadmin needs this:** Scripts run from any working directory — always use absolute paths in `mkdir -p`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Created in the wrong place | `pwd` first, or always use absolute paths in scripts |

---

### Task 13 — `install -d` as an alternative

**Purpose:** When you need to set mode AND owner AND group in one shot (the `install` command).

```bash
sudo install -d -m 0750 -o root -g root /srv/installed
ls -ld /srv/installed
```

**Expected output:**

```
drwxr-x---. 2 root root 6 Sep 12 16:25 /srv/installed
```

**Switches**

| Token | Meaning |
|---|---|
| `install -d DIR` | Create directory (the `d` flag means "directory mode") |
| `-m MODE` | Permission mode |
| `-o USER` | Owner |
| `-g GROUP` | Group |

**Output decoded**

| Token | Meaning |
|---|---|
| `drwxr-x---.` | Mode 0750 set in one command |
| Owner+group = root | Set explicitly |

> `install -d` replaces `mkdir -p` + `chmod` + `chown` for repetitive tasks. It's not in every distro by default but is standard on RHEL/CentOS/Rocky/Alma via coreutils.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `install: invalid option -- 'd'` | Use the three-step pattern from Task 11 instead |

---

### Task 14 — Detect what `mkdir -p` actually created

**Purpose:** Use `touch /tmp/mark` + `find -newer` from Lab 07 to audit.

```bash
touch /tmp/before-mark
mkdir -pv ~/mkdir-lab/audit/{logs,configs}/{prod,dev}
find ~/mkdir-lab/audit -newer /tmp/before-mark -type d
```

**Expected output:**

```
mkdir: created directory '/home/ec2-user/mkdir-lab/audit'
mkdir: created directory '/home/ec2-user/mkdir-lab/audit/logs'
mkdir: created directory '/home/ec2-user/mkdir-lab/audit/logs/prod'
mkdir: created directory '/home/ec2-user/mkdir-lab/audit/logs/dev'
mkdir: created directory '/home/ec2-user/mkdir-lab/audit/configs'
mkdir: created directory '/home/ec2-user/mkdir-lab/audit/configs/prod'
mkdir: created directory '/home/ec2-user/mkdir-lab/audit/configs/dev'
/home/ec2-user/mkdir-lab/audit
/home/ec2-user/mkdir-lab/audit/logs
/home/ec2-user/mkdir-lab/audit/logs/prod
/home/ec2-user/mkdir-lab/audit/logs/dev
/home/ec2-user/mkdir-lab/audit/configs
/home/ec2-user/mkdir-lab/audit/configs/prod
/home/ec2-user/mkdir-lab/audit/configs/dev
```

**Switches**

| Token | Meaning |
|---|---|
| `touch /tmp/before-mark` | Sentinel file with current timestamp |
| `find -newer FILE` | Find entries newer than FILE (Lab 14) |
| `-type d` | Directories only |

**Output decoded**

| Block | Meaning |
|---|---|
| `mkdir: created ...` lines | What `-v` told us |
| `find` listing | Independent verification |

**Why on RHCA RH342:** During recovery, prove which directories your scripts actually touched.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `find` returns nothing | Sentinel was AFTER the `mkdir` — re-run with sentinel first |

---

### Task 15 — `mkdir -p` for kubelet/etcd paths (CKA)

**Purpose:** Recreate the canonical Kubernetes control-plane directories after a reset.

```bash
sudo mkdir -pv -m 0700 /etc/kubernetes/pki/etcd
sudo mkdir -pv -m 0755 /var/lib/kubelet/pods
sudo mkdir -pv -m 0700 /var/lib/etcd
sudo mkdir -pv -m 0755 /etc/cni/net.d

sudo ls -ld /etc/kubernetes/pki/etcd /var/lib/kubelet /var/lib/etcd /etc/cni/net.d
```

**Expected output (excerpt):**

```
mkdir: created directory '/etc/kubernetes'
mkdir: created directory '/etc/kubernetes/pki'
mkdir: created directory '/etc/kubernetes/pki/etcd'
mkdir: created directory '/var/lib/kubelet'
mkdir: created directory '/var/lib/kubelet/pods'
mkdir: created directory '/var/lib/etcd'
mkdir: created directory '/etc/cni'
mkdir: created directory '/etc/cni/net.d'
drwx------. 2 root root  6 Sep 12 16:30 /etc/kubernetes/pki/etcd
drwxr-xr-x. 3 root root 17 Sep 12 16:30 /var/lib/kubelet
drwx------. 2 root root  6 Sep 12 16:30 /var/lib/etcd
drwxr-xr-x. 2 root root  6 Sep 12 16:30 /etc/cni/net.d
```

**Switches**

| Token | Meaning |
|---|---|
| `-p` | Build the whole chain |
| `-v` | Show what was created |
| `-m 0700` | Owner-only access for sensitive paths (PKI, etcd) |
| `-m 0755` | World-readable for paths the kubelet shares with CNI |

**Output decoded**

| Path | Mode | Used for |
|---|---|---|
| `/etc/kubernetes/pki/etcd` | 0700 | etcd member certificates — secrets |
| `/var/lib/kubelet/pods` | 0755 | Kubelet's pod state |
| `/var/lib/etcd` | 0700 | etcd data — strictly private |
| `/etc/cni/net.d` | 0755 | CNI config — readable by kubelet |

**Why on CKA:** Kubelet pre-flight checks fail if these directories are missing — recovery starts here.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Kubelet still failing | Check ownership too — most should be `root:root` |

---

### Task 16 — Idempotent `mkdir -p` in shell scripts

**Purpose:** The pattern every production script uses.

```bash
cat > ~/mkdir-lab/setup.sh <<'EOF'
#!/bin/bash
set -euo pipefail

BASE="${BASE:-/srv/myapp}"
DIRS=( logs configs backup/daily backup/weekly )

for d in "${DIRS[@]}"; do
  mkdir -pv -m 0750 "$BASE/$d"
done

ls -ld "$BASE" "${DIRS[@]/#/$BASE/}"
EOF
chmod +x ~/mkdir-lab/setup.sh
BASE=~/mkdir-lab/app1 ~/mkdir-lab/setup.sh
```

**Expected output (excerpt):**

```
mkdir: created directory '/home/ec2-user/mkdir-lab/app1'
mkdir: created directory '/home/ec2-user/mkdir-lab/app1/logs'
mkdir: created directory '/home/ec2-user/mkdir-lab/app1/configs'
mkdir: created directory '/home/ec2-user/mkdir-lab/app1/backup'
mkdir: created directory '/home/ec2-user/mkdir-lab/app1/backup/daily'
mkdir: created directory '/home/ec2-user/mkdir-lab/app1/backup/weekly'
drwxr-x---. 5 ec2-user ec2-user ... /home/ec2-user/mkdir-lab/app1
...
```

**Switches**

| Token | Meaning |
|---|---|
| `set -euo pipefail` | Strict-mode bash: exit on error, undefined vars, pipe failures |
| `${BASE:-/srv/myapp}` | Default value if `BASE` unset |
| `DIRS=( ... )` | Array of subdirectories |
| `${DIRS[@]/#/$BASE/}` | Expand each element with `$BASE/` prefix |

**Output decoded**

| Line | Meaning |
|---|---|
| Each `created directory` | One subdirectory built |
| Final `ls -ld` rows | Verification |

**Why a sysadmin needs this on RHCE EX294:** This is the bash equivalent of `ansible.builtin.file: state=directory` with a loop — recognize the pattern in both worlds.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `unbound variable` | `set -u` is strict — make sure all variables are initialized |

---

### Task 17 — Ansible role scaffolding pattern

**Purpose:** Create the standard Ansible role layout in one line.

```bash
ROLE=~/mkdir-lab/roles/myapp
mkdir -pv "$ROLE"/{tasks,handlers,templates,files,vars,defaults,meta}
find "$ROLE" -type d | sort
```

**Expected output:**

```
mkdir: created directory '/home/ec2-user/mkdir-lab/roles'
mkdir: created directory '/home/ec2-user/mkdir-lab/roles/myapp'
mkdir: created directory '/home/ec2-user/mkdir-lab/roles/myapp/tasks'
mkdir: created directory '/home/ec2-user/mkdir-lab/roles/myapp/handlers'
mkdir: created directory '/home/ec2-user/mkdir-lab/roles/myapp/templates'
mkdir: created directory '/home/ec2-user/mkdir-lab/roles/myapp/files'
mkdir: created directory '/home/ec2-user/mkdir-lab/roles/myapp/vars'
mkdir: created directory '/home/ec2-user/mkdir-lab/roles/myapp/defaults'
mkdir: created directory '/home/ec2-user/mkdir-lab/roles/myapp/meta'
/home/ec2-user/mkdir-lab/roles/myapp
/home/ec2-user/mkdir-lab/roles/myapp/defaults
/home/ec2-user/mkdir-lab/roles/myapp/files
/home/ec2-user/mkdir-lab/roles/myapp/handlers
/home/ec2-user/mkdir-lab/roles/myapp/meta
/home/ec2-user/mkdir-lab/roles/myapp/tasks
/home/ec2-user/mkdir-lab/roles/myapp/templates
/home/ec2-user/mkdir-lab/roles/myapp/vars
```

**Switches**

| Token | Meaning |
|---|---|
| `{tasks,handlers,...}` | Brace expansion → 7 subdirectories |
| `find ... -type d \| sort` | Verify with sorted directory list |

**Output decoded**

| Path | Ansible purpose |
|---|---|
| `tasks/` | Main task list (`main.yml`) |
| `handlers/` | Restart triggers |
| `templates/` | Jinja2 templates |
| `files/` | Static files for `copy` |
| `vars/` | High-precedence variables |
| `defaults/` | Low-precedence variables |
| `meta/` | Role dependencies and metadata |

**Why on RHCE EX294:** First step of every role — `ansible-galaxy init` does the same thing, but knowing the manual command means you can fix layout drift.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ansible-galaxy` not installed | Manual `mkdir -p` works just as well |

---

### Task 18 — `mkdir -p` with explicit SELinux context

**Purpose:** Force a specific context, not just the policy default.

```bash
sudo mkdir -pv -Z --context=system_u:object_r:httpd_sys_content_t:s0 /srv/web-special
sudo ls -ldZ /srv/web-special
```

**Expected output:**

```
mkdir: created directory '/srv/web-special'
drwxr-xr-x. 2 root root system_u:object_r:httpd_sys_content_t:s0 6 Sep 12 16:40 /srv/web-special
```

**Switches**

| Token | Meaning |
|---|---|
| `-Z` | Enable SELinux context setting |
| `--context=CTX` | Use this explicit context |
| `httpd_sys_content_t` | Apache content type |

**Output decoded**

| Token | Meaning |
|---|---|
| `httpd_sys_content_t` | Now labeled for Apache — even though `/srv` would default to something else |

**Why on RHCSA Task 16:** Sometimes the task says "create the directory with context X" — `-Z --context=` does it in one step.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Context not applied | Your kernel may lack the type — try `chcon -t TYPE` afterwards |

---

### Task 19 — RHCSA-style scenario

**Task statement:** *"Create `/srv/web/sites/intranet/htdocs` and `/srv/web/sites/intranet/logs`, both owned by `apache:apache`, mode `0750`. Verify."*

```bash
sudo useradd apache -m -r 2>/dev/null || true
sudo groupadd -r apache 2>/dev/null || true

sudo mkdir -pv -m 0750 /srv/web/sites/intranet/{htdocs,logs}
sudo chown -R apache:apache /srv/web/sites/intranet
sudo restorecon -Rv /srv/web

ls -ldZ /srv/web/sites/intranet/htdocs /srv/web/sites/intranet/logs
stat -c '%U %G %a %C %n' /srv/web/sites/intranet/htdocs /srv/web/sites/intranet/logs
```

**Expected output (excerpts):**

```
mkdir: created directory '/srv/web'
mkdir: created directory '/srv/web/sites'
mkdir: created directory '/srv/web/sites/intranet'
mkdir: created directory '/srv/web/sites/intranet/htdocs'
mkdir: created directory '/srv/web/sites/intranet/logs'
drwxr-x---. 2 apache apache system_u:object_r:var_t:s0 6 Sep 12 16:45 /srv/web/sites/intranet/htdocs
drwxr-x---. 2 apache apache system_u:object_r:var_t:s0 6 Sep 12 16:45 /srv/web/sites/intranet/logs
apache apache 750 system_u:object_r:var_t:s0 /srv/web/sites/intranet/htdocs
apache apache 750 system_u:object_r:var_t:s0 /srv/web/sites/intranet/logs
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `useradd/groupadd` | Ensure user/group exist |
| `mkdir -pv -m 0750` | Create the full chain with correct mode for the leaves |
| `chown -R` | Apply ownership to the entire intranet subtree |
| `restorecon -Rv` | Recompute SELinux contexts from policy |
| `ls -ldZ` + `stat -c` | One-line audit per directory |

**Output decoded**

| Token | Meaning |
|---|---|
| `apache apache` | Owner+group correct |
| `750` | Mode correct |
| `var_t` | SELinux type from policy for `/srv` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `useradd` reports user exists | Fine — `\|\| true` suppresses the failure |
| Intermediate `/srv/web` and `/srv/web/sites` have mode `0755` | Expected: `-m` only applies to leaves with `-p` |

---

### Task 20 — CKA-style scenario

**Task statement (CKA-flavored):** *"After a node reset, recreate `/etc/kubernetes/pki/etcd`, `/var/lib/kubelet/pods`, and `/var/lib/etcd` with correct permissions (700 for PKI/etcd, 755 for kubelet). Audit."*

```bash
sudo mkdir -pv -m 0700 /etc/kubernetes/pki/etcd /var/lib/etcd
sudo mkdir -pv -m 0755 /var/lib/kubelet/pods

stat -c '%a %U %G %C %n' /etc/kubernetes/pki/etcd /var/lib/etcd /var/lib/kubelet/pods
```

**Expected output:**

```
mkdir: created directory '/etc/kubernetes'
mkdir: created directory '/etc/kubernetes/pki'
mkdir: created directory '/etc/kubernetes/pki/etcd'
mkdir: created directory '/var/lib/etcd'
mkdir: created directory '/var/lib/kubelet'
mkdir: created directory '/var/lib/kubelet/pods'
700 root root system_u:object_r:cert_t:s0 /etc/kubernetes/pki/etcd
700 root root system_u:object_r:var_lib_t:s0 /var/lib/etcd
755 root root system_u:object_r:var_lib_t:s0 /var/lib/kubelet/pods
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| First `mkdir -pv -m 0700` | Sensitive PKI and etcd paths — owner-only |
| Second `mkdir -pv -m 0755` | Kubelet shares with CNI/runtime — world-readable |
| `stat -c` | One-line audit per path |

**Output decoded**

| Path | Mode | Why |
|---|---|---|
| `/etc/kubernetes/pki/etcd` | 700 | Stores etcd member certs — protect from other users |
| `/var/lib/etcd` | 700 | Stores etcd database — strictly private |
| `/var/lib/kubelet/pods` | 755 | Kubelet's pod state — needs to be traversable |

**Cleanup**

```bash
cd ~
rm -rf ~/mkdir-lab
rm -f /tmp/before-mark
sudo rm -rf /srv/installed /srv/web /srv/web-special /var/www/html/site2 /tmp/absolute
# Do NOT clean /etc/kubernetes or /var/lib/{etcd,kubelet} on a live node!
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Kubelet still complains about missing dir | Verify owner is `root:root`, mode matches expectations |

---

## 🔍 `mkdir` Decision Guide

```
Single dir, parents exist?           → mkdir <dir>
Nested path?                         → mkdir -p <a/b/c>
Idempotent in a script?              → mkdir -p (safe to re-run)
Need specific mode?                  → mkdir -m MODE <dir>      (only last dir with -p)
Need mode everywhere in the tree?    → loop:  for d in a a/b a/b/c; do mkdir -m M -p "$d"; done
                                       OR    mkdir -p TREE && chmod -R M TREE
Need owner+group too?                → mkdir + chown   OR   install -d -m M -o U -g G DIR
Need SELinux context?                → mkdir -p -Z [--context=CTX] DIR
Want to verify what got created?     → add -v
Need many subdirs?                   → mkdir -p base/{a,b,c/{x,y},d}
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Set up `~/mkdir-lab`
- [ ] 02 `mkdir` a single directory
- [ ] 03 `mkdir` multiple in one command
- [ ] 04 Plain `mkdir a/b/c` fails on missing parents
- [ ] 05 `mkdir -p a/b/c/d/e` builds the full chain
- [ ] 06 `mkdir -p` is idempotent (no error on second run)
- [ ] 07 Brace expansion with `-p` for complex trees
- [ ] 08 `mkdir -pv` to see what was created
- [ ] 09 `mkdir -m MODE` for permissions at creation
- [ ] 10 `mkdir -p -Z` for SELinux context at creation
- [ ] 11 Combine `mkdir + chown + chmod + ls -ldZ`
- [ ] 12 Absolute vs relative paths
- [ ] 13 `install -d -m -o -g` as alternative
- [ ] 14 Audit creations with `find -newer`
- [ ] 15 Recreate kubelet/etcd/PKI directories (CKA)
- [ ] 16 Script-friendly idempotent loop
- [ ] 17 Ansible role scaffolding in one command
- [ ] 18 Explicit SELinux context with `--context=CTX`
- [ ] 19 Exam: `/srv/web/sites/intranet/{htdocs,logs}` with mode 0750
- [ ] 20 CKA: recreate `/etc/kubernetes/pki/etcd`, `/var/lib/etcd`, `/var/lib/kubelet/pods`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Plain `mkdir` on nested path | `No such file or directory` | Add `-p` |
| `mkdir -m MODE -p` and surprised intermediates are 0755 | `-m` only affects leaves | Loop or use `chmod -R` afterwards |
| Forgetting `-p` in scripts | Second run errors | Always use `-p` for idempotency |
| `mkdir /var/www/html/X` without `restorecon` | Apache may 403 on contents later | Use `mkdir -p -Z` or `restorecon -Rv` |
| `mkdir -p .` or `..` | Confusing but harmless | Use real paths |
| Brace expansion in `sh`/`dash` | Literal `{a,b}` directory created | Switch to bash |
| Forgetting to `chown` after `sudo mkdir` | Files owned by root, your user can't write | Always `chown` to the service user |

---

## 📌 Exam Strategy

**RHCSA EX200**
- Task 8 patterns: `sudo mkdir -pv -m MODE PATH && sudo chown OWNER:GROUP PATH && ls -ldZ PATH`.
- Use `install -d -m -o -g` when you remember it — saves three commands.
- Always end with `ls -ldZ` (or `stat -c '%U %G %a %C %n'`) to verify all four attributes.

**RHCE EX294 (Ansible)**
- `ansible.builtin.file: state=directory` parameters map directly to `mkdir -p` + `chown` + `chmod` + SELinux flags:
  - `path:`, `state: directory`
  - `mode:`, `owner:`, `group:`
  - `setype:`, `seuser:`, `serole:`
  - `recurse: yes` for nested

**CKA**
- Memorize these paths and their modes:
  - `/etc/kubernetes/pki/{,etcd}` → 0700
  - `/var/lib/etcd` → 0700
  - `/var/lib/kubelet/pods` → 0755
  - `/etc/cni/net.d` → 0755
- `sudo mkdir -p -m 0700` is the fastest recovery start.

**RHCA**
- RH342: rebuild service trees during recovery — `mkdir -p` + `restorecon` + service restart.
- RH358: every service has its own tree under `/etc/<svc>` and `/var/lib/<svc>`.
- RH236: brick paths under `/data/gluster/brickN/` — `mkdir -p` + `restorecon`.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Navigation | `pwd` and `cd` after `mkdir -p` |
| Lab 06 — `ls -lZ` | Verify mode, owner, group, context |
| Lab 07 — `touch` | Often follows `mkdir -p` |
| Lab 08 — `cp -a` | Deploy content into the new directory |
| Lab 10 — `mv` | Move existing content into the new path |
| Lab 11 — `rm`/`rmdir` | Symmetric cleanup operation |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
