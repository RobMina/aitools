# Running pyutils Analyses Off-Cluster (Laptop venv)

Reference for running pyutils/uproot analyses on a laptop or workstation instead of a
Mu2e interactive node (GPVM). Load this only when the user wants to analyze locally.

## Why this works

pyutils and the EventNtuple analysis stack are ROOT-free at the Python level: all file
I/O goes through `uproot` and `awkward`. They install with `pip` on Linux, macOS (Intel
or Apple silicon) without CVMFS, `mu2einit`, `pyenv ana`, or a ROOT installation.

What does **not** run off-cluster: the art/Offline chain (simulation, digitization,
reconstruction, EventNtuple production). Those use CVMFS-distributed Linux binaries and
stay on the cluster. Off-cluster work starts from an existing EventNtuple `.root` file.

## Decision rule: local vs cluster

The deciding factor is **data locality**, not compute.

| Run locally (venv) | Run on the cluster |
|---|---|
| Post-processing, plotting, and cut studies on ntuples you already have | Full-sample streaming over many files, or inputs of tens of GB that are not worth copying |
| Development and debugging on one file or a small sample | First-time ntuple production (needs the art/Offline chain) |
| Any analysis whose inputs are already local or worth fetching once | Jobs that must read dCache/tape through `use_remote=True` |

Rules of thumb:

1. Fetch only the ntuples you need. For a development loop, one file is usually enough.
2. Develop and debug locally on that file, then run the same script unchanged on the
   cluster for the full sample.
3. If outputs (histogram caches, JSON, merged results) are already local, re-plotting
   needs no cluster access at all.

## Create a project venv

Use a dedicated virtual environment per project, and always call its interpreter
explicitly.

```bash
python3 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install uproot awkward numpy scipy matplotlib pandas vector
.venv/bin/pip install "git+https://github.com/Mu2e/pyutils.git"
```

The `pip install git+https://github.com/Mu2e/pyutils.git` line is the install command
from the pyutils README for use on your own computer. For development on pyutils itself,
clone the repository and run `.venv/bin/pip install -e .` from inside it (see
[Setup and Installation](../SKILL.md#setup-and-installation)).

Verify:

```bash
.venv/bin/python3 -c "import pyutils, uproot, awkward; print('ok')"
```

### Never install into the system Python

- Do not run `pip install` (or `sudo pip`, `pip --user`) against the system interpreter.
- Run scripts as `.venv/bin/python3 script.py`, not `python3 script.py`.
- Installing into the project venv with `.venv/bin/pip install <pkg>` is fine.
- If a package is missing and you are not sure which environment it belongs in, report the
  `ImportError` to the user and ask, instead of installing anything.

## Fetch the input data

Copy the needed EventNtuple file(s) from the cluster to a local data directory using
whatever transfer method your site provides (`scp`/`rsync` from a GPVM, `ifdh cp`, or
`mdh copy-file`). Keep large `.root` files out of the git repository and out of
documentation trees.

## Read EventNtuple with uproot (no ROOT, no gallery)

Do not write ROOT or gallery macros to inspect EventNtuple contents. Read the tree
with uproot and analyze with awkward:

```python
import uproot

with uproot.open("nts.owner.description.version.sequence.root") as f:
    tree = f["EventNtuple/ntuple"]
    print(tree.num_entries)
    arrays = tree.arrays(["evt.run", "evt.subrun", "evt.event"], library="ak")
```

Notes:

- Use `tree.keys()` to list branches before requesting them.
- Use `tree.num_entries` for event counts rather than parsing logs.
- Nested `vector<vector<T>>` branches (for example `trksegs`) may be unsplit and read
  back as whole records. Read the branch, then index its fields in awkward, rather than
  asking uproot for `trksegs.sid` directly.
- Prefer vectorized awkward/numpy expressions over per-event Python loops. They are
  often an order of magnitude faster on jagged data.

## Run pyutils processors on local files

pyutils works unchanged on local files. Point the file list at local paths and keep
`use_remote=False` (see [Common Configuration Options](../SKILL.md#common-configuration-options)).
Parallel `jobs > 1` also works locally. Set it to a value below your core count.

Only remote streaming through dCache/mdh (`use_remote=True`) requires cluster
access and its authentication and CVMFS tooling.

## Keep analysis code in a package, not inline Python

- Put analysis code in importable modules or scripts under version control, and run them
  with `.venv/bin/python3 path/to/script.py`.
- Avoid pasting long ad-hoc `python - <<EOF` snippets into the shell. They cannot be
  reviewed, re-run, or reused.
- Make input and output paths command-line arguments (`argparse`). Do not hard-code
  cluster paths such as `/exp/...` or `/pnfs/...` as the only default, since they do not
  exist on a laptop.
- If sibling modules use flat imports (`import common`), run with the scripts directory
  on the path: `PYTHONPATH=path/to/scripts .venv/bin/python3 path/to/scripts/run.py`.
- Reuse and extend an existing script before writing a new one.

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `ModuleNotFoundError: pyutils` | Not installed in the venv, or the system `python3` was used. Run `.venv/bin/python3`, then `.venv/bin/pip install "git+https://github.com/Mu2e/pyutils.git"`. |
| `FileNotFoundError` on `/exp/...` or `/pnfs/...` | The script's default path is a cluster path. Pass an explicit local `--ntuple-dir` / `--output-dir`. |
| Script tries to fetch or authenticate | `use_remote` is True. Set `use_remote=False` and use local files. |
| Branch not found | Branch names differ between EventNtuple versions. List with `tree.keys()`. |
