# Diagnostics

Reusable functions for computing physical quantities from model output.

These are **importable building blocks**, not standalone scripts. Each module
exposes functions that take `xarray.Dataset` objects (or numpy arrays) as input
and return computed quantities — no plotting, no file I/O beyond what the
caller provides.

## Convention

- Functions accept `xarray.Dataset` or raw arrays as their primary input.
- Return values are numpy arrays, scalars, or small `xarray.DataArray` objects.
- No side effects: no plotting, no file writing, no global state.
- Import from analysis scripts in `runs/<name>/` or `analysis/intercomparisons/`:

```python
from diagnostics.mixed_layer_depth import compute_mld
from diagnostics.heat_content import compute_heat_content
```

## Planned diagnostics

- `mixed_layer_depth.py` — MLD from T/S threshold or density criterion
- `heat_content.py` — Ocean heat content integrated over depth
- `overturning.py` — Meridional overturning streamfunction
