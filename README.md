# legoESM ocean-experiments ledger (template)

This is a **template repository** — click **"Use this template"** to create your own
personal experiment ledger for ocean simulations with
[legoESM](https://github.com/climate-federation/legoESM), run via
[legoESM-ocean-runners](https://github.com/lesommer/legoESM-ocean-runners).

It is your **personal lab notebook**: it records *what was run, where, how, and what
came of it* — not the model code, not the runners, and not the output data.

## How this repo is organized

- **`runs/`** — one subdirectory per experiment, self-contained. Each contains the
  declaration (`run.yaml`), resolved config (`config.yaml`), provenance record
  (`experiment.tag`), and analysis scripts. Names follow
  `<nickname>_<YYYY-MM-DD>_<description>`. See `runs/README.md`.
- **`machines/`** — personal machine configurations (scheduler settings, paths,
  devices). Copy `_template.yaml` per machine; auto-detect with
  `bin/lego-detect-machine`.
- **`analysis/`** — reusable diagnostic functions (`diagnostics/`) and
  cross-experiment intercomparison scripts (`intercomparisons/`).
- **`reports/`** — dated, shareable experiment-series reports.
- **`bin/`** — CLI helpers (`run-experiment`, `lego-detect-machine`,
  `setup-workspace`, `check-workspace`).

## Quick start

Prerequisites: sibling checkouts of [`legoESM`](https://github.com/climate-federation/legoESM)
and [`legoESM-ocean-runners`](https://github.com/lesommer/legoESM-ocean-runners)
side-by-side with this repo, plus a Python environment with JAX and legoESM installed.

### 1. Configure your machine(s)

```bash
bin/lego-detect-machine --write    # first-guess YAML; edit nickname, paths, env_setup
```

### 2. Declare an experiment (locally)

```bash
mkdir -p runs/<nickname>_<date>_my-first-experiment
cat > runs/<nickname>_<date>_my-first-experiment/run.yaml <<EOF
template: ocean/baroclinic_gyre   # runners-owned templates: legoESM-ocean-runners/templates/
                                  # graduated templates: ocean/<stem> (resolved from legoESM)
machine: my-machine                       # optional — auto-detected if omitted
overrides:
  time.duration_days: 30
status: planned
EOF
git add runs/ && git commit -m "declare <nickname>_<date>_my-first-experiment" && git push
```

### 3. Initialize and run (on the target machine)

```bash
git pull
bin/run-experiment init runs/<nickname>_<date>_my-first-experiment/   # generates config.yaml, experiment.tag, run.sh
bin/run-experiment run  runs/<nickname>_<date>_my-first-experiment/   # direct, or submit run.sh to your scheduler
```

### 4. Record and analyze

```bash
# update status in run.yaml to completed/failed; write notes.md
git add runs/<nickname>_<date>_my-first-experiment/ && git commit -m "..." && git push
# locally: git pull, then write analysis scripts next to the run (use paths from experiment.tag)
```

## Reproducibility chain

```
run.yaml (intent) → config.yaml (resolved) → experiment.tag (provenance)
```

`experiment.tag` pins the exact runner tag and legoESM commit; reproduce any
experiment with `bin/run-experiment reproduce runs/<name>/`.

## Rules

- Never commit model output (NetCDF, Zarr, checkpoints) — only provenance metadata
- Never commit notebook outputs (nbstripout pre-commit hook is configured)
- Never edit `../legoESM/` or `../legoESM-ocean-runners/` from this repo

See `AGENTS.md` for the full conventions.
