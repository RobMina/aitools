---
name: submitting-grid-jobs
description: Practical pitfalls when submitting Mu2e grid jobs directly with jobsub_submit (jobsub_lite). Use when writing or debugging a jobsub_submit script, when ifdh cp / gfal-copy back to /pnfs scratch fails with 403 Forbidden or Permission refused, when deciding --need-storage-modify, --tar-file-name, --disk or the number of jobs, when a muse tarball is rebuilt or re-uploaded on every submit, when a submit seems to hang after "Submitting N jobs", or when deciding between the grid and interactive GPVM nodes. Not for mu2eprodsys production submission.
compatibility: Requires a Mu2e interactive node (mu2egpvm or build node), valid Kerberos ticket and Fermilab token, jobsub_lite, and a muse build (muse tarball)
metadata:
  version: "1.0.0"
  last-updated: "2026-10-14"
---

# Submitting Grid Jobs with jobsub_submit

Pitfalls that cost real submissions when sending your own analysis or reco jobs to the
grid with `jobsub_submit`. The `mu2eprodsys` wrapper (mu2egrid) handles most of this for
you; see [building-with-muse](../building-with-muse/SKILL.md) for that route. This skill
covers the direct `jobsub_submit` route. Dataset/file inputs via SAM are in
[finding-data-sam](../finding-data-sam/SKILL.md).

Check flag spellings against `jobsub_submit --help` on your installed version.

## Checklist before every submission

1. Is the work worth the grid at all? (see "Grid or interactive node?")
2. Tarball built **once**, path passed to the submit script.
3. `--need-storage-modify=/mu2e/...` present if any job writes to `/pnfs`.
4. Jobs sized to ~30 min each from a measured seconds/file; `--disk` scaled to match.
5. Worker/driver scripts survive `set -e` (see "Shell gotchas").

## 1. Build the muse tarball once

`muse tarball` is expensive (about 5 minutes for a full Offline build). Do not call it
from inside a submit script that you run repeatedly.

```bash
cd /path/to/your/muse/workspace      # directory with the muse build
muse setup
muse tarball                         # prints the Code.tar.bz2 path
```

- Rebuild only when the Offline code, FHiCL, or bundled scripts change.
- Record the printed path and pass it to the submit script as an argument (for example
  `submit.sh /path/to/Code.tar.bz2 ...`). Submit scripts should require an existing
  tarball path rather than build one.
- Extra files can be bundled: `muse tarball *.txt data/*.root`
  (see [building-with-muse](../building-with-muse/SKILL.md)).
- Run `jobsub_submit` from a scratch/data directory, not from an app area or a synced
  working tree: jobsub writes temporary tarball copies into the current directory.

## 2. Grant write scope: `--need-storage-modify`

Jobs submitted with `jobsub_submit` get a reduced token. By default it is read-only for
`/mu2e` storage (plus job staging). Copy-out of outputs to dCache scratch then fails with
errors such as `HTTP 403 Permission refused (destination)` from `gfal-copy`, or
`403 Forbidden` / `File exists` from `ifdh cp`, even though the directory exists and has
correct permissions.

Request write scope explicitly, using the **logical `/mu2e/...` path**, not the `/pnfs`
FUSE path:

```bash
jobsub_submit -G mu2e --role=Analysis \
  --need-storage-modify=/mu2e/scratch/users/<user>/<outdir> \
  ...
```

| | Path |
|---|---|
| Correct | `--need-storage-modify=/mu2e/scratch/users/<user>/<outdir>` |
| Wrong | `--need-storage-modify=/pnfs/mu2e/scratch/users/<user>/<outdir>` |

Notes:

- The directory should cover everything the job writes (the flag takes directories needing
  `storage.modify` scope in the job token).
- `jobsub_submit` cannot infer the scope from an output path held in a shell variable
  inside the job script; always pass the flag on the command line.
- When adapting someone else's `jobsub_submit` command (for example a production
  validation script run under a production role), add the flag if your jobs write to
  scratch.
- Diagnose a 403 by running one small test job that prints the scopes of the job token
  (decode the token file named by `$BEARER_TOKEN_FILE` inside the job), not by resubmitting
  the full campaign.

## 3. Size jobs to about 30 minutes

Each job pays fixed overhead: the tarball upload (minutes, first submit only), queue wait,
and setup on the worker (`source setupmu2e-art.sh`, `setup ifdhc`; tens of seconds).
Very short jobs spend a large share of their time on overhead, while very long jobs lose
more work when one fails. Target about 30 minutes of execution per job.

Size from a **measured** rate, not a round number. Run a handful of files interactively
and time them:

```
n_jobs = ceil(n_files * seconds_per_file / (30 * 60))
files_per_job = ceil(n_files / n_jobs)
```

- Re-measure for each input type; data and MC (or different file sizes) differ.
- Scale `--disk` with files per job when inputs are staged to the worker:
  `disk >= files_per_job * MB_per_file` plus output and headroom. A flat `--disk=5GB` is
  fine for a few files and fails outright with ~100 staged files. Jobs that exceed their
  requested disk, memory or lifetime go into a held state.
- Prefer streaming inputs or fewer files per job over very large `--disk` requests; a big
  disk request matches fewer worker slots and the jobs may sit idle in the queue.
- Typical resource flags: `--memory=2500MB --disk=10GB --expected-lifetime=1h` (check the
  defaults of your jobsub_lite version).

## 4. Grid or interactive node?

Decide the machine **before** sizing jobs. If the work is seconds per file (for example a
vectorised Python/ntuple analysis), run it on the interactive GPVMs, one process per core
you are entitled to, scattering file batches across a few nodes. The grid overhead
(tarball, queue, worker setup) dwarfs such work, and the 30-minute packing rule would force
you to stage many files per job.

| Cost per file | Route |
|---|---|
| Seconds | Interactive nodes, batches of files |
| Minutes, or needs many cores/long jobs | Grid, sized per section 3 |

Warning sign: if the derived `--disk` is above roughly 10 GB, the job is staging a lot to
do little; reconsider the route rather than requesting more disk. Re-ask the question
after speeding up an analysis; the right answer can change.

## 5. Do not re-upload the tarball with `dropbox://`

`--tar-file-name dropbox://<path>/Code.tar.bz2` (the underscore spelling `--tar_file_name`
is also accepted; `tardir://<dir>` tars a directory first) copies the tarball to RCDS/CVMFS and
unpacks it on the worker (location in `$INPUT_TAR_DIR_LOCAL`).

Observed in the jobsub_lite client source (`lib/tarfiles.py`): on **every** submit it first
rewrites the whole tarball to `./<name><pid>.tbz2` in the current directory (single-threaded
bz2, setting modes to at least 0755), hashes that copy, and only then checks whether the
content is already on RCDS. An unchanged tarball is not re-uploaded, but a large muse
tarball (hundreds of MB) can still cost many minutes of apparent hang after
`Submitting N jobs...`.

Once a tarball is published, resubmit with the RCDS directory itself:

```bash
jobsub_submit ... \
  --tar-file-name /cvmfs/fifeuser<N>.opensciencegrid.org/sw/mu2e/<hash> \
  ...
```

- The `<hash>` path is printed in the first job's `.out` file.
- A plain path (no `dropbox://` prefix) is passed through as is, and the worker unpacks
  it the same way.
- For the **first** upload, submit from a data directory (see section 1) and clean up any
  leftover `*.tbz2` after an interrupted submit.
- Do not assume the hash equals the SHA-256 of your original file; the hash is of the
  rewritten copy.

## 6. Shell gotchas under `set -e`

Driver and worker scripts commonly use `set -e`; two setup steps return non-zero even
though the environment is fine.

```bash
set -e
source /cvmfs/mu2e.opensciencegrid.org/setupmu2e-art.sh || true
cd /path/to/your/muse/workspace      # directory containing the muse build
muse setup || true
```

- `source setupmu2e-art.sh` can return non-zero (an internal `return 1` when the
  production DB is not configured), which aborts the script before `mu2e` is available.
- `muse setup` can exit non-zero in a fresh non-interactive shell because of an
  `export` of a variable name containing a hyphen
  (`export: 'MU2E-TRIG-CONFIG_INC=...' not a valid identifier`). The environment is still
  set; treat the exit code as noise.
- `muse setup` must be run from the directory that contains the muse build (the build
  workspace, where the `.muse` file lives), not from your FCL or script directory.
  Otherwise `mu2e` does not get added to `PATH`.
- On the worker, after the tarball is unpacked, set up with `source Code/setup.sh` (or
  `muse setup Code`) and use `|| true` there as well.

## Quick diagnosis

| Symptom | Likely cause | Fix |
|---|---|---|
| `403` / `Permission refused (destination)` on copy-out to `/pnfs` | Read-only job token | Add `--need-storage-modify=/mu2e/scratch/users/<user>/<dir>` (section 2) |
| Same 403 despite the flag | Path given as `/pnfs/mu2e/...` | Use the logical `/mu2e/...` prefix |
| Submit sits silent after "Submitting N jobs" | `dropbox://` retarring a large tarball | Resubmit with the `/cvmfs` RCDS path (section 5) |
| Script dies right after `source setupmu2e-art.sh` or `muse setup` | `set -e` with non-zero return | Append `|| true` (section 6) |
| `mu2e` not found after `muse setup` | Ran from wrong directory | `cd` to the muse build workspace first |
| Jobs held | Exceeded `--disk`, `--memory` or `--expected-lifetime` | Re-measure and resize (section 3) |
| Jobs idle, never start | Very large `--disk` request | Fewer files per job, stream, or use interactive nodes |
