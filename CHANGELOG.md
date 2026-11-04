# Changelog

## 0.1.0 – PyPSA >= 1.0 compatibility

Validated with PyPSA 1.4.0, pandas 3.1, xarray 2026.9, numpy 2.5, linopy 0.10
(22 tests passing). The package layout and public API are unchanged.

### Packaging
- `setup.py`: version `0.0.8` -> `0.1.0`; requires `pypsa >= 1.0` and `linopy`.
- `environment.yml`: `pypsa` -> `pypsa>=1.0`.

### `netallocation/plot.py`
- `projected_area_factor` is imported from `pypsa.geo.get_projected_area_factor`
  (falls back to `pypsa.plot` for old versions).
- `pypsa.descriptors.Dict` -> `pypsa.common.Dict` (with fallback).
- `n.snapshot_weightings[0]` -> `n.snapshot_weightings['generators'].iloc[0]`
  (weightings are a DataFrame with one column per use).

### `netallocation/utils.py`
- `snapshot_weightings`: always returns a `DataArray` over `snapshot`, using
  the `generators` column if the weightings are a DataFrame.
- `set_default_if_none`: the components `Process` and `ShuntImpedance`
  (new in PyPSA) are dropped from defaults when empty. All uses of
  `n.branch_components` / `n.one_port_components` in this module now go through
  `check_branch_comps` / `check_one_port_comps`.
- `check_carriers`: newer PyPSA defines an empty `carrier` column on all
  components; empty/NaN entries are now filled with the old defaults (`Load`,
  or the bus carrier) instead of only adding missing columns.
- `reindex_by_bus_carrier`: builds the (bus, carrier) MultiIndex with
  `MultiIndex.from_arrays` instead of renaming with tuples (not supported by
  current pandas/xarray), and no longer mutates its input.

### `netallocation/grid.py`
- `Incidence`: the matrix is built directly from `bus0`/`bus1` instead of
  `n.incidence_matrix`, whose branch ordering differs in PyPSA 1.x (this gave
  silently wrong PTDFs). Output is identical to the old behaviour.
- `Cycles`: `nx.OrderedGraph` (removed in networkx 3) -> `nx.Graph`; uses
  `check_branch_comps`.
- `impedance`: reindexing uses `reindex_like`; component selection of links
  goes through the new helper `_select_component`, and the link pseudo
  impedance is written positionally, as the current xarray does not support
  the previous MultiIndex-based selection/assignment.
- `_one_port_attr`: uses `check_one_port_comps`.
- `energy_production`: uses `utils.snapshot_weightings`.

### `netallocation/linalg.py`
- `pinv` and `inv` (dense, sparse and `pre_clean`) pass `dims` explicitly so
  that the transposed dimension names are kept with current xarray.

### `netallocation/cost.py`
- Cycle-constraint (Kirchhoff) shadow prices: networks solved with
  `n.optimize()` do not populate `n.sub_networks_t.mu_kirchhoff_voltage_law`.
  New helpers `_has_kvl_shadowprices` and `_cycle_constraint_price_from_model`
  read the duals and coefficients of the `Kirchhoff-Voltage-Law` constraint
  from the linopy model. `cycle_constraint_price`, `cycle_constraint_cost` and
  `congestion_revenue` use them when the legacy attribute is absent; the
  legacy `lopf` path is unchanged. Duals are only available if solving with
  `n.optimize(assign_all_duals=True)`; otherwise a warning is logged.
- `nodal_co2_price`: constraint names are cast to `str` before `.str`
  matching; the CO2 dual is taken as `abs()` for `<=` constraints because the
  sign convention of duals changed.
- `n.passive_branch_components` -> `check_passive_branch_comps`.

### `netallocation/flow.py`
- `marginal_welfare_contribution`: rewritten. The old version used
  `n.lopf`, pyomo/gurobi_persistent and a non-existent `.opf` module. The new
  one runs `n.optimize` for the full network and once per removed line
  (`solver_name='highs'` by default, extra kwargs forwarded) and returns the
  generator profits (or networks). Signature changed: `formulation` removed,
  `solver_name` and `**kwargs` added. **Not tested.**

### Tests
- `test_cost.py`: `n.lopf(...)` -> `n.optimize(solver_name='highs',
  assign_all_duals=True)`; the `ac_dc` network is loaded with its stored link
  `p_set` removed (PyPSA 1.x fixes flows to `p_set`); `snapshot_weightings`
  uses the `generators` column; branch cost series are reindexed over all
  branches.
- `test_grid.py`: `test_cycles` compares the span of the cycle bases to
  `n.cycle_matrix()` (the basis is not unique) instead of `pypsa.pf.find_cycles`.

### Known remaining items
- `n.df(...)`/`n.pnl(...)` are deprecated in PyPSA 1.x (removal in 2.0).
- Pandas 4 deprecation warnings originate from PyPSA itself.
