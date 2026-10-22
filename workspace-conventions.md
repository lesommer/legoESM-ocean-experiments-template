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
| `legoESM/` | The climate model (JAX-based, differentiable ESM) + its own reproducibility spine (templates, init, `legoesm run`, per-run `run_manifest.json`, `legoesm reproduce`) | github.com/climate-federation/legoESM |
| `legoESM-ocean-runners/` | Thin ocean delta layer over that spine: curated recipe×setup templates, the patch chain (pin → tag → patches → env reconstruction), the ocean data catalog, fleet ops | github.com/lesommer/legoESM-ocean-runners |
| experiment ledger (this repo) | Personal experiment ledger (declare runs, record provenance, analyze) | — |

### Data flow

```
legoESM  ←pinned by commit──  legoESM-ocean-runners  ←templates+overrides──  experiment ledger
  (model + reproducibility        (thin ocean delta layer:                         (lab notebook)
   spine: templates, init,        curated recipes/setups, patches,
   run, manifest, reproduce)     data catalog, machines, receipts)
```

- **legoESM** is never edited from the other repos. Patches are the only coupling mechanism.
- **legoESM-ocean-runners** pins a specific legoESM commit in `legoESM_version.txt` and provides curated templates + runners. It does **not** duplicate legoESM's in-repo reproducibility spine: the per-run `run_manifest.json` written by legoESM drivers is designed upstream as a **superset of `experiment.tag`**, and `scripts/reproduce.py` reconstructs the environment (runner tag → pin → patches → data) that the manifest only *describes*.
- **The experiment ledger** references templates by name, adds per-run overrides, and records provenance. When a run writes a `run_manifest.json`, cite its `state_digest`/versions in `notes.md` (the manifest is the run-time record; `experiment.tag` is the init-time anchor).

## 2. legoESM Key Paths

legoESM has its own short `AGENTS.md` (agent orientation) and `CLAUDE.md` (the full
house rulebook). The repo uses a uv-workspace federation layout — ocean code lives
under `packages/ocean/legoesm/ocean/`, core under `packages/core/legoesm/`, the
coupler/driver under `packages/coupler/legoesm/`. Key paths for reference:

- `packages/coupler/legoesm/driver/config.py` — `ExperimentConfig` (atmosphere)
- `packages/coupler/legoesm/driver/coupled_config.py` — `CoupledConfig`
- `src/legoesm/config.py` / `src/legoesm/experiment_registry.py` — template adapters, `legoesm run` dispatch
- `packages/ocean/legoesm/ocean/state.py` — Ocean state containers + ocean config types
- `packages/ocean/legoesm/ocean/recipes.py` — **the recipe catalog** (`get_recipe`, `assemble_ocean_config`)
- `packages/ocean/legoesm/ocean/experiments/recipe_map.py` — **recipe×setup tags** (`EXPERIMENT_RECIPES`, verified by tests)
- `packages/ocean/legoesm/ocean/experiments/dino.py` — DINO cases + `DINO_RECIPES` cards
- `packages/ocean/legoesm/ocean/fidelity/` — NEMO-match recipe cards (`nemo_match_recipe.py`, ...)
- `packages/ocean/legoesm/ocean/dynamics/` — Ocean dynamics modules (4 grids)
- `packages/ocean/legoesm/ocean/physics/` — Vertical/lateral mixing, bottom drag, convection, ice shelf
- `packages/ocean/legoesm/ocean/experiments/` — Idealized test cases
- `packages/ocean/legoesm/ocean/forcing/` — JRA55-do, WOA, Dai-Trenberth, SSS restoring
- `packages/ocean/legoesm/ocean/coupler/` — Ice-shelf, OMIP2, runoff, tidal mixing applicators
- `packages/core/legoesm/constants.py` — Authoritative physical constants (never inline)
- `packages/core/legoesm/thermo.py` — Saturation thermodynamics (never re-derive)
- `packages/core/legoesm/io/git_provenance.py` + `packages/coupler/legoesm/driver/restart.py` — `run_manifest.json` writing (the manifest is a superset of `experiment.tag`)
- `scripts/run/run_omip.py` / `run_omip_core2.py` — OMIP runners
- `scripts/matrix/run_ocean_test_matrix.py` — Ocean test matrix (owns the validated per-case configs)
- `scripts/experiment/init_experiment.py`, `fetch_data.py`, `validate_templates.py` — upstream's in-repo spine (downstream ergonomics, absorbed)
- `config/templates/` — upstream's own template library (generic templates graduate here; merge-and-remove, like patches)
- `config/data_catalog.yaml` — upstream data catalog (the runners keep the richer ocean catalog)

## 3. Cross-Repo Boundary Rules

These rules span all three repos and cannot be enforced by any single one.

### NEVER

- **Never edit legoESM source code** from the runners or experiments repos. If a model change is needed, work in `legoESM/` on a branch, then create a patch in `legoESM-ocean-runners/patches/`.
- **Never hardcode physical constants** — always use `legoESM/packages/core/legoesm/constants.py`.
- **Never re-derive thermodynamics** — always use `legoESM/packages/core/legoesm/thermo.py`.
- **Never declare an unanchored `ocean.recipe`** — recipe names are fidelity claims; a declaration must resolve in the pinned legoESM (recipe_map tag, DINO card, or the tripole lane identity).
- **Never duplicate numerics** — ocean baroclinic helpers go in `ocean_tendency_common.py`, barotropic in `barotropic_common.py`.
- **Never break autodiff/JIT** — end-to-end `jax.grad` compatibility is a primary design goal.
- **Never store data files** (NetCDF, Zarr, etc.) in any repo. Use `data_catalog.yaml` to reference data by ID; fetch with `fetch_data.py`.
- **Never commit experiment output** (results, logs, checkpoints) — only provenance metadata (`experiment.tag`).
- **Never commit notebook outputs** — `nbstripout` pre-commit hook strips them.

### ALWAYS

- **Always use templates** from `legoESM-ocean-runners/templates/` when declaring experiments.
- **Always name recipes by catalog name** (`ocean.recipe: <name>` from the pinned legoESM recipe catalog / recipe_map / DINO cards) — never re-spell scheme selections when a named recipe exists.
- **Always record `experiment.tag`** after initialization — it's the reproducibility anchor (the run-time `run_manifest.json` is its superset, where a lane writes one).
- **Always use `apply_overrides()`** with dot-notation for parameter changes (never edit resolved `config.yaml` by hand).
- **Always reference data by catalog ID** — never hardcode paths.
- **Always number patches** as `NNN-description.patch` (applied alphabetically).
- **Always remove patches** once merged upstream into legoESM.
- **Always validate templates** after changing `legoESM_version.txt` (run `lego-validate`).
- **Always update `project_status.md`** in the runners repo when changing runner capabilities or template test status.

## 4. Experiment Lifecycle

### Step 1: Declare (locally, in the experiment ledger)

```bash
mkdir -p runs/laptop_2026-06-01_baroclinic-gyre
cat > runs/laptop_2026-06-01_baroclinic-gyre/run.yaml <<EOF
template: ocean/baroclinic_gyre
machine: my-machine            # optional — auto-detected via bin/lego-detect-machine if omitted
overrides:
  grid.resolution: 128x256
  ocean.A_h: 1e6
  time.duration_days: 730
status: planned
EOF
git add runs/laptop_2026-06-01_baroclinic-gyre/run.yaml
git commit -m "add laptop_2026-06-01_baroclinic-gyre experiment" && git push
```

### Step 2: Initialize (on HPC or locally)

```bash
git pull
bin/run-experiment init runs/laptop_2026-06-01_baroclinic-gyre/
# Generates: config.yaml, experiment.tag, run.sh
# Updates status to "submitted"
# --output-dir is required (set automatically by bin/run-experiment)
```

Or directly via runners:
```bash
cd ../legoESM-ocean-runners
python scripts/init_experiment.py ocean/baroclinic_gyre \
    --name laptop_2026-06-01_baroclinic-gyre \
    --output-dir /path/to/output \
    -o grid.resolution=128x256 \
    -o ocean.A_h=1e6 \
    -o time.duration_days=730
```

### Step 3: Ensure data is available

```bash
cd ../legoESM-ocean-runners
python scripts/fetch_data.py check ocean/baroclinic_gyre
# For realistic experiments:
python scripts/fetch_data.py fetch global_ocean/omip_jra55
```

### Step 4: Run

```bash
bin/run-experiment run runs/laptop_2026-06-01_baroclinic-gyre/
# Or: cd runs/laptop_2026-06-01_baroclinic-gyre && bash run.sh
# Or: submit run.sh to SLURM/PBS scheduler
```

### Step 5: Record

```bash
# Update status in run.yaml to completed or failed
# Write notes.md with observations
git add runs/laptop_2026-06-01_baroclinic-gyre/ && git commit -m "laptop_2026-06-01_baroclinic-gyre: completed" && git push
```

### Step 6: Analyze (locally)

```bash
git pull
# Write analysis scripts in runs/laptop_2026-06-01_baroclinic-gyre/ referencing output paths from experiment.tag
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
| `ocean.recipe` (matrix lanes) | `ocean/experiments/recipe_map.py` | `EXPERIMENT_RECIPES` — the case's verified recipe tag |
| `ocean.recipe` (dino) | `ocean/experiments/dino.py` | `DINO_RECIPES` cards (`run_dino.py --recipe`) |
| `ocean.recipe` (tripole OMIP) | `ocean/fidelity/nemo_match_recipe.py` | `NEMOMatchTripoleRecipeConfig` (catalog: `omip_nemo_match_tripole_v1`) |
| (recipe catalog) | `ocean/recipes.py` | `LATLON_RECIPES` / `MPAS_RECIPES`, `get_recipe`, `assemble_ocean_config` |
| `physics.vertical_mixing.*` | `ocean/physics/vertical_mixing/config.py` | `KPPConfig` / `RichardsonConfig` |
| `physics.lateral_mixing.*` | `ocean/physics/lateral_mixing/config.py` | `GMRediConfig` |
| `physics.bottom_drag.*` | `ocean/physics/bottom_drag/config.py` | `BottomDragConfig` |
| `physics.surface_forcing.*` | `ocean/physics/surface_forcing/config.py` | Forcing config |
| `coupled.*` | `driver/coupled_config.py` | `CoupledConfig` presets |
| `time.*` | `timestepping/` | Time integrator selection |
