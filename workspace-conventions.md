# Workspace Conventions — lego-world

This document describes the cross-repo architecture and shared conventions for the
three-repo legoESM experiment system. It is stored in the experiment-ledger repo
and symlinked to the workspace root as `AGENTS.md` (via `bin/setup-workspace`).

For repo-specific rules, see the per-repo `AGENTS.md` files:
- `legoESM-ocean-runners/AGENTS.md` — template format, runners, patches, data catalog
- experiment-ledger `AGENTS.md` — experiment ledger, machine configs, analysis, reports

## 1. Workspace Layout

Three Git repositories live side-by-side under a shared workspace directory:

| Repo | Role | URL |
|------|------|-----|
| `legoESM/` | The climate model (JAX-based, differentiable ESM) | github.com/climate-federation/legoESM |
| `legoESM-ocean-runners/` | Experiment templates, runners, patches, data catalog | github.com/lesommer/legoESM-ocean-runners |
| experiment ledger (this repo) | Personal experiment ledger (declare runs, record provenance, analyze) | — |

### Data flow

```
legoESM  ←pinned by commit──  legoESM-ocean-runners  ←templates+overrides──  experiment ledger
  (model)                       (runners)                                       (lab notebook)
```

- **legoESM** is never edited from the other repos. Patches are the only coupling mechanism.
- **legoESM-ocean-runners** pins a specific legoESM commit in `legoESM_version.txt` and provides templates + runners.
- The experiment ledger references templates by name, adds per-run overrides, and records provenance.

## 2. legoESM Key Paths

legoESM does not have its own `AGENTS.md`. Key paths for reference:

- `src/legoesm/driver/config.py` — `ExperimentConfig` (atmosphere)
- `src/legoesm/driver/coupled_config.py` — `CoupledConfig`
- `src/legoesm/ocean/state.py` — Ocean state containers + ocean config types
- `src/legoesm/ocean/dynamics/` — Ocean dynamics modules (4 grids)
- `src/legoesm/ocean/physics/` — Vertical/lateral mixing, bottom drag, convection, ice shelf
- `src/legoesm/ocean/experiments/` — Idealized test cases
- `src/legoesm/ocean/forcing/` — JRA55-do, WOA, Dai-Trenberth, SSS restoring
- `src/legoesm/ocean/coupler/` — Ice-shelf, OMIP2, runoff, tidal mixing applicators
- `src/legoesm/constants.py` — Authoritative physical constants (never inline)
- `src/legoesm/thermo.py` — Saturation thermodynamics (never re-derive)
- `scripts/run_omip.py` — OMIP runner (all grids)
- `scripts/run_ocean_test_matrix.py` — Ocean test matrix
- `config/` — YAML configurations for built-in experiments
- `docs/ocean_experiments_reference.md` — Reference for all ocean experiments

## 3. Cross-Repo Boundary Rules

These rules span all three repos and cannot be enforced by any single one.

### NEVER

- **Never edit legoESM source code** from the runners or experiments repos. If a model change is needed, work in `legoESM/` on a branch, then create a patch in `legoESM-ocean-runners/patches/`.
- **Never hardcode physical constants** — always use `legoESM/src/legoesm/constants.py`.
- **Never re-derive thermodynamics** — always use `legoESM/src/legoesm/thermo.py`.
- **Never duplicate numerics** — ocean baroclinic helpers go in `ocean_tendency_common.py`, barotropic in `barotropic_common.py`.
- **Never break autodiff/JIT** — end-to-end `jax.grad` compatibility is a primary design goal.
- **Never store data files** (NetCDF, Zarr, etc.) in any repo. Use `data_catalog.yaml` to reference data by ID; fetch with `fetch_data.py`.
- **Never commit experiment output** (results, logs, checkpoints) — only provenance metadata (`experiment.tag`).
- **Never commit notebook outputs** — `nbstripout` pre-commit hook strips them.

### ALWAYS

- **Always use templates** from `legoESM-ocean-runners/templates/` when declaring experiments.
- **Always record `experiment.tag`** after initialization — it's the reproducibility anchor.
- **Always use `apply_overrides()`** with dot-notation for parameter changes (never edit resolved `config.yaml` by hand).
- **Always reference data by catalog ID** — never hardcode paths.
- **Always number patches** as `NNN-description.patch` (applied alphabetically).
- **Always remove patches** once merged upstream into legoESM.
- **Always validate templates** after changing `legoESM_version.txt` (run `lego-validate`).
- **Always update `project_status.md`** in the runners repo when changing runner capabilities or template test status.

## 4. Experiment Lifecycle

### Step 1: Declare (locally, in the experiment ledger)

```bash
mkdir -p runs/<nickname>_<date>_baroclinic-gyre
cat > runs/<nickname>_<date>_baroclinic-gyre/run.yaml <<EOF
template: 3d_idealized/baroclinic_gyre
machine: my-machine         # optional — auto-detected via bin/lego-detect-machine if omitted
overrides:
  grid.resolution: 128x256
  ocean.A_h: 1e6
  time.duration_days: 730
status: planned
EOF
git add runs/<nickname>_<date>_baroclinic-gyre/run.yaml
git commit -m "add <nickname>_<date>_baroclinic-gyre experiment" && git push
```

### Step 2: Initialize (on HPC or locally)

```bash
git pull
bin/run-experiment init runs/<nickname>_<date>_baroclinic-gyre/
# Generates: config.yaml, experiment.tag, run.sh
# Updates status to "submitted"
# --output-dir is required (set automatically by bin/run-experiment)
```

Or directly via runners:
```bash
cd ../legoESM-ocean-runners
python scripts/init_experiment.py 3d_idealized/baroclinic_gyre \
    --name <nickname>_<date>_baroclinic-gyre \
    --output-dir /path/to/output \
    -o grid.resolution=128x256 \
    -o ocean.A_h=1e6 \
    -o time.duration_days=730
```

### Step 3: Ensure data is available

```bash
cd ../legoESM-ocean-runners
python scripts/fetch_data.py check 3d_idealized/baroclinic_gyre
# For realistic experiments:
python scripts/fetch_data.py fetch global_ocean/omip_jra55
```

### Step 4: Run

```bash
bin/run-experiment run runs/<nickname>_<date>_baroclinic-gyre/
# Or: cd runs/<nickname>_<date>_baroclinic-gyre && bash run.sh
# Or: submit run.sh to your scheduler (SLURM/PBS/OAR)
```

### Step 5: Record

```bash
# Update status in run.yaml to completed or failed
# Write notes.md with observations
git add runs/<nickname>_<date>_baroclinic-gyre/ && git commit -m "<nickname>_<date>_baroclinic-gyre: completed" && git push
```

### Step 6: Analyze (locally)

```bash
git pull
# Write analysis scripts in runs/<nickname>_<date>_baroclinic-gyre/ referencing output paths from experiment.tag
```

## 5. Synchronization Protocol

### Local → HPC (git-based)

1. **Local**: Create `runs/<nickname>_<date>_<description>/run.yaml` (status: planned), commit, push
2. **HPC**: `git pull`, run `bin/run-experiment init runs/<nickname>_<date>_<description>/`, submit job
3. **HPC**: Update status (completed/failed), commit provenance, push
4. **Local**: `git pull`, review `experiment.tag` + `notes.md`, write analysis

### Runners sync with legoESM

```bash
cd legoESM-ocean-runners
bash scripts/sync_legoesm.sh --tag   # updates legoESM_version.txt, validates, tags
```

### Override for feature branches

Create `legoESM_version.txt.override` (git-ignored) with alternative `ref:` and `commit:` to work with a non-main legoESM branch.

## 6. legoESM Configuration Mapping

Template YAML fields map to legoESM Python objects:

| Template field | legoESM module | Class |
|---------------|----------------|-------|
| `experiment.mode` | `driver/config.py` | Mode dispatch |
| `grid.*` | `driver/config.py` | `GridConfig` |
| `atmosphere.*` | `driver/config.py` | `ExperimentConfig` fields |
| `ocean.*` | `ocean/state.py` | Ocean config NamedTuples |
| `physics.vertical_mixing.*` | `ocean/physics/vertical_mixing/config.py` | `KPPConfig` / `RichardsonConfig` |
| `physics.lateral_mixing.*` | `ocean/physics/lateral_mixing/config.py` | `GMRediConfig` |
| `physics.bottom_drag.*` | `ocean/physics/bottom_drag/config.py` | `BottomDragConfig` |
| `physics.surface_forcing.*` | `ocean/physics/surface_forcing/config.py` | Forcing config |
| `coupled.*` | `driver/coupled_config.py` | `CoupledConfig` presets |
| `time.*` | `timestepping/` | Time integrator selection |
