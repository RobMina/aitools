---
name: running-on-gpvms
description: Shell and session gotchas for non-interactive work on the Mu2e interactive GPVM nodes. Use when launching background or long jobs over ssh or tmux, when a job "launches fine" but produces nothing, on "cannot overwrite existing file" (noclobber, >|), set -u aborting muse setup, SIGHUP killing nohup jobs, expired Kerberos tickets or "Key has expired" in tmux, stat/ls on /pnfs hanging, pyenv ana not found, hadd or root not found, Ceph quota or data-area placement, or how many jobs to run per GPVM.
compatibility: Requires ssh access to a Mu2e interactive GPVM node (Kerberos), the /cvmfs/mu2e.opensciencegrid.org setup scripts, and optionally tmux
metadata:
  version: "1.0.0"
  last-updated: "2026-10-16"
---

# Running on the GPVMs

Practical gotchas for scripted or non-interactive work (ssh one-liners, driver
scripts, tmux sessions) on the shared Mu2e interactive nodes (`<gpvm>`). Most
cost time because they fail **silently**: the launch returns, nothing runs, and
there is no error. If a job "launches fine" but produces nothing, check this
list in order.

## 1. Shell basics

### Linux, POSIX tools

The GPVMs run Linux. Use `root` (not `root.exe`) and standard POSIX tools and
paths.

### Your shell may set `noclobber`

Mu2e's example `.bashrc` sets `noclobber` for interactive shells, and many
personal setups set it everywhere. Then `>` onto an existing file fails with
`cannot overwrite existing file`, and a compound command containing it exits
non-zero without running the rest.

- In commands: use `>|` (force overwrite), `>>` (append), or `rm -f FILE` first.
- In scripts: add `set +o noclobber` after the environment setup, because
  internal `: > log` or `cat > wrapper.fcl` redirects hit the same wall.

```bash
my_job --args >| run.log 2>&1
```

### `set -u` goes after environment setup

`setupmu2e-art.sh`, `muse setup` and Spack setup scripts reference unset
variables. `set -u` before them aborts the whole shell, and the error vanishes
if setup output is sent to `/dev/null`. Source the setup first, then enable
`set -u`.

### `muse setup` under `set -e`

Muse versions before v4_16_00 could print
``export: `MU2E-TRIG-CONFIG_INC=...' not a valid identifier`` (hyphenated
package name) and return non-zero even though the environment was set up
correctly; newer versions convert the hyphen. `|| true` is harmless either way
in scripts using `set -e`:

```bash
muse setup || true
```

### `pyenv ana` is a shell function

`pyenv` on the GPVMs is a shell function defined by the Mu2e setup script, not
the standalone pyenv version manager. It does not exist until the script is
sourced, so source and activate in the same invocation:

```bash
source /cvmfs/mu2e.opensciencegrid.org/setupmu2e-art.sh && pyenv ana
```

Afterwards `python3` has the analysis stack (`uproot`, `awkward`, `numpy`,
`matplotlib`, `scipy`, `pyutils`). The system Python on the nodes lacks these.
A fresh non-login shell does not re-source the setup for you, so repeat both
commands in each new one-shot shell that needs the stack.

### Which setup puts `hadd` and `root` on PATH

`hadd` and `root` come from the art/ROOT Spack environment that `muse setup`
loads, not from the base setup script or the analysis Python environment:

| after `source setupmu2e-art.sh` and ... | `hadd` / `root` |
|---|---|
| nothing else | not on PATH |
| `pyenv ana` | not on PATH |
| `muse setup ops` | not on PATH |
| `muse setup` in a Muse working directory | on PATH |
| `muse setup SimJob` (any published release) | on PATH |

To merge ntuples from a `pyenv ana` shell, open a separate shell with
`muse setup` (or `muse setup SimJob`) for the `hadd`, rather than sourcing a
ROOT `thisroot.sh` by hand. Checked on an AL9 GPVM, 2026-10-16.

### FHiCL `#include` does not take absolute paths

`#include "/abs/path/base.fcl"` fails with `search_path ... Can't find file`.
FHiCL resolves includes against `FHICL_FILE_PATH`. Prepend the directory and
include by relative name:

```bash
export FHICL_FILE_PATH="$FCLDIR:$FHICL_FILE_PATH"
# wrapper.fcl:  #include "base.fcl"
```

## 2. Backgrounding jobs over ssh

A one-shot `ssh <gpvm> "cmd &"` has two failure modes.

**The job dies (SIGHUP).** When the ssh channel closes, the remote login shell
exits and tears down its process group. A plain backgrounded job can be killed
before it writes anything: the launch returns 0, `ps` shows nothing, and the
log stays 0 bytes. Use `nohup` or a persistent tmux session.

**The ssh call blocks until the job ends.** `ssh h "cd X && nohup job &"`
backgrounds the whole `&&` list in a subshell that keeps ssh's stdout/stderr
open. `</dev/null` alone does not fix it. Background only the job, in its own
subshell, with all three fds redirected:

```bash
ssh <gpvm> "cd /path/to/workdir && ( nohup bash job.sh </dev/null >| job.log 2>&1 & )"
```

Alternatively, start the job inside a tmux session on the node
(`tmux new -d -s NAME 'cmd >| log 2>&1'`) and poll the log. A tmux session
outlives the ssh connection.

When several nodes should run in parallel, launch each with the `( ... & )`
form. If only one node starts for minutes while the others wait, the blocking
form above is the cause, not a hang.

When a launch looks stuck, verify with `pgrep -af` on the node and `tail` the
log before relaunching: the job may be running, and a duplicate launch breaks
the one-job-per-host rule below.

## 3. Kerberos and long-lived sessions

On the GPVMs the Kerberos ticket is forwarded on each ssh login into a **new**
cache file (`/tmp/krb5cc_<uid>_*`). It is a static copy that expires on its own
schedule, not renewed in place. A long-lived tmux session (or ssh master
connection) keeps pointing at the cache that existed when it started.

Consequences:

- A session older than the ticket lifetime can hold an **expired** ticket while
  a fresh login on the same node reports a healthy one.
- Symptoms: `Key has expired` from the home directory (`/nashome/...`) or
  `.bashrc`, and tmux panes that die before running anything.
- A stale tmux server on a node also hands its dead credentials to every new
  session it spawns.

Fixes:

1. Before a multi-hour job, kill and recreate the tmux session (and any ssh
   master connection) so it inherits the current cache.
2. Confirm **inside that session** with `klist | grep krbtgt`. A separate
   one-shot ssh uses a different cache and can report a healthy ticket for a
   session that is dead.
3. If new sessions on a node keep dying with `Key has expired`, stop the stale
   tmux server there (`tmux kill-server`); it respawns clean on next use. As an
   immediate workaround, use another GPVM.
4. `k5push <node>` (see the Authentication wiki page) pushes your current ticket
   to another node. It is not confirmed to refresh a cache that an already
   running tmux session holds, so prefer restarting the session.

## 4. /pnfs access

`/pnfs` is a dCache namespace backed by a database. `find`, `ls -l`/`ls -lr`
and `stat` fetch the full record of every file and are slow and expensive; a
plain `ls` reads only the directory record (see the Dcache wiki page). Many
nodes alias `ls` to `ls --color=auto`, which is slower; use `\ls` to bypass the
alias.

- In scripts, list or check existence with `ifdh ls <path>` (or `ifdh ll`), or use
  `samweb locate-file` / metacat for dataset queries.
- Copy with `ifdh cp`, or read via xrootd.
- art/`mu2e` jobs read their input art files straight from the `/pnfs` path
  (`mu2e -s /pnfs/.../file.art`, `-n N` to limit events). Do not prestage inputs
  to `/tmp` for interactive runs.

See [finding-data-sam](../finding-data-sam/SKILL.md) for catalogue queries.

## 5. Where data goes, and quota

- **App area** (`/exp/mu2e/app/users/<user>/`): code, build products, scripts.
  No data files.
- **Data area** (`/exp/mu2e/data/users/<user>/`) or `/pnfs/mu2e/scratch/users/<user>/`:
  ntuples, art files, plots, logs, test outputs. This applies to small test runs
  too.
- **Inputs are not data-area content.** Do not copy digi/raw/art inputs into the
  data area to reuse them across jobs. Stage per job into the node's local
  `/tmp`, process, and delete.

The `/exp` areas are CephFS with per-directory quotas. Hitting one does not
produce a clear error: art aborts on a ROOT write, driver scripts lose their
exit lines, and the job looks like an external kill. Before a multi-GB run,
compare free quota with the expected output size; if a job dies with nothing in
its log, check quota first.

```bash
d=/exp/mu2e/data/users/<user>
getfattr --only-values -n ceph.dir.rbytes "$d"          # bytes used
getfattr --only-values -n ceph.quota.max_bytes "$d"     # quota
```

`mu2eQuota` (see the Disks wiki page) prints the same summary. Use these rather
than `du`, which is slow on CephFS. Usage accounting lags
deletes by a few tens of seconds. Intermediate `.art` files accumulate quickly
when a test driver does not delete them; remove them once the ntuples they
produced are done.

## 6. One heavy job per host

The GPVMs are shared quad-core interactive machines meant for short to moderate
jobs (see the ComputingLogin wiki page). No per-user core limit is published; as
etiquette, run **one** heavy job (one mu2e job, one core) at a time per node.

- Parallelise by spreading jobs across several GPVMs, one job each, and run each
  node's list sequentially. Do not background several chains on one node.
- Ad hoc checks count: a `file_info_dumper`, a python/uproot entry count, ROOT,
  or a heavy `grep` on a node already running a job is a second core, and can
  push ssh into banner or connection timeouts. Run them on an idle node.
- Builds go on the build node (`mu2ebuild02`), not a GPVM.

## Checklist for a job that "launched but did nothing"

1. `pgrep -af <name>` on the node: is it running, or already gone?
2. Does the log exist and have bytes? A 0-byte log suggests a `noclobber`
   failure or a SIGHUP kill (sections 1, 2).
3. Was `set -u` placed before setup, or (with older Muse) `muse setup` run under
   `set -e` without `|| true`?
4. `klist | grep krbtgt` from inside the session; `Key has expired` in the pane
   points at a stale session (section 3).
5. Free space in the target quota (section 5).
6. Wrapper FCL includes use relative names with `FHICL_FILE_PATH` set.
