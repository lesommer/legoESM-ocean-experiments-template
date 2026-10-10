# How to Set Up and Run Experiments

This is the **operational guide** for setting up legoESM on a machine and running
ocean experiments from declaration to analysis. It is written so that either you
or a less-familiar LLM can follow it end-to-end without guessing.

For full repository conventions, provenance rules, and boundary policies, see
`AGENTS.md`. This document focuses on *how to do it*, not *why the rules exist*.

---

## 1. One-time workspace setup

```bash
# 1. Clone the three repos side-by-side
mkdir -p ~/lego-world && cd ~/lego-world
git clone https://github.com/climate-federation/legoESM.git
git clone https://github.com/lesommer/legoESM-ocean-runners.git
# clone your experiment ledger (from your own fork of this template):
git clone <your-ledger-url>.git

# 2. Install legoESM (uv workspace)
cd legoESM
uv sync --extra dev
# For MPI runs additionally: uv pip install -r requirements_mpi.txt

# 3. Verify
python -c "import jax; print(jax.default_backend(), jax.devices())"
python -c "import legoesm; print('legoESM OK')"
```

Then link the workspace-level conventions file and check the layout:

```bash
cd ../<your-ledger>
bin/setup-workspace      # creates AGENTS.md symlink at the workspace root
bin/check-workspace      # validates repos, symlink, and version pin alignment
```

## 2. Register the machine

```bash
bin/lego-detect-machine            # inspect what is detected
bin/lego-detect-machine --write    # write machines/<hostname>.yaml
```

Edit the generated YAML (`machines/<hostname>.yaml`):

- `nickname` — used in experiment directory names (keep it short)
- `scheduler` — `null` (direct execution), `slurm`, `oar`, `pbs`, or `lsf`
- `data_root`, `legoesm_dir`, `output_root` — filesystem paths for this machine
- `env_setup` — commands to activate the environment (module loads, conda/venv)

See `machines/_template.yaml` for the full schema with comments.

## 3. Declare an experiment (locally)

```bash
mkdir -p runs/<nickname>_<date>_<description>
cat > runs/<nickname>_<date>_<description>/run.yaml <<EOF
template: 3d_idealized/baroclinic_gyre
machine: <machine>          # optional — auto-detected if omitted
overrides:
  time.duration_days: 30
status: planned
EOF
git add runs/<nickname>_<date>_<description>/run.yaml
git commit -m "declare <nickname>_<date>_<description>" && git push
```

Pick the template from `../legoESM-ocean-runners/templates/`.

## 4. Initialize (on the target machine)

```bash
git pull
bin/run-experiment init runs/<nickname>_<date>_<description>/
# Generates: config.yaml, experiment.tag, run.sh
```

Ensure input data is available (for realistic experiments):

```bash
cd ../legoESM-ocean-runners
python scripts/fetch_data.py check <template>
python scripts/fetch_data.py fetch <template>   # if needed
```

## 5. Run

```bash
bin/run-experiment run runs/<nickname>_<date>_<description>/
```

- **No scheduler** (`scheduler: null`): runs directly in your activated environment.
- **SLURM/PBS**: the runner submits `run.sh` and prints the job id; monitor with
  `squeue`/`qstat`.
- **OAR**: `run.sh` carries `#OAR` directives; submit with
  `cd runs/<name> && oarsub -S ./run.sh`.

## 6. Record

```bash
# Update status in run.yaml to completed or failed
# Write notes.md with observations
git add runs/<nickname>_<date>_<description>/ && git commit -m "<name>: completed" && git push
```

## 7. Analyze (locally)

```bash
git pull
# Write analysis scripts in the run directory; locate output data via
# experiment.tag (never hardcode machine-specific absolute paths).
```

## 8. Reproduce an experiment

```bash
bin/run-experiment reproduce runs/<nickname>_<date>_<description>/
# Checks out the runner tag + legoESM commit recorded in experiment.tag,
# applies patches, and re-runs.
```
