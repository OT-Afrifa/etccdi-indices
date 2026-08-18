# Branch: francis-percentile-indices

Adds the percentile half of the ETCCDI set, four fixed-threshold indices,
two extra output frequencies (monthly and DJF seaosn) and a user-facing 
"compute one metric" notebook, covering two roadmap items in the 
README(extra metrics; monthly-resolution indices).

## Contents

- `multimodel_etccdi.py` : one module unifying all four models (CESM2-WACCM6,
  UKESM1-1, MIROC-ES2H, E3SMv3) behind a single API: per-model loaders in native
  hub layouts, day-of-year percentile thresholds (SSP2-4.5 2020-2039, 5-day window,
  wet-day masking for precipitation), the Lee et al. three-way warming/SAI/combined
  decomposition, Welch significance, and a dataset writer targeting the shared S3 layout.
  
- `compute_any_index.ipynb` : press-go notebook where you choose an index, model(s),
  scenario, members, and frequency in one cell. You get the field back with a
  cost-guidance table, a Cartopy quick-look map, an optional write to S3, and a recipe
  for adding new indicators via `INDEX_REGISTRY`.

## What this branch adds to the existing indices module

**Twelve more indices-** The eight percentile indices (`R95p`, `R99p`, `TX90p`, `TX10p`,
`TN90p`, `TN10p`, `WSDI`, `CSDI`) plus four fixed-threshold ones: `RX1D` and `RX5D` (1-day and 5-day precipitation maxima, following Cindy Wang's definitions and short names so the two index
sets merge without renaming), `GSL` (growing season length) and `GDD` (growing degree days).

Fixed-threshold indices need no baseline percentile, so they run far faster and keep all four models.

**Output frequency-** `SUPPORTED_FREQ = ('YS', 'MS', 'QS-DEC')`: annual, monthly, and the DJF season. The frequency threads through `compute_index_for_members`, `run_comparison`, `run_lee_framework` and both cached wrappers, and appears in the comparison cache filenames so a DJF run cannot silently load an annual result. The tag is empty for `YS`, so caches written before the argument existed are still found. 

**Hemisphere-aware growing season-** `growing_season_length_global` wraps xclim's `growing_season_length`: northern gridpoints use a 1 July mid-date on a calendar year, southern gridpoints a 1 January mid-date one a July-to-June year, relabelled to January stamps. Without it, southern values are pinned by the mid-date constraint rather than by temperature (a validation on real data gave 181 days for every year at a Patagonian point, which is exactly 1 Jnauary to 1 July on a 365-day calendar).

**Registry attributes-** Entries may carry an `attrs` dict supplying `long_name` and `comment`, filled per computation from `extra_kwargs` so a field computed at a non-default threshold is labelled with the threshold actually used. xclim's `indices` layer sets `units` and nothing else, so without this a released file carries no human-readable name.

**Subtropical comparison configs-** `subtropical` and `subtropical_combined` give the 2->3 and 1->3 Lee contrasts for G6-1.5K-SAI. The existing `sai` key carries G6-1.5K-HiLLA despite its name, so before this the subtropical scenario was covered only by the direct SAI-versus-HiLLA contrast. Currently, its labels are scenario-explicit.

**Assessment window argument-** `run_lee_framework_cached` takes `assessment`, so all four models can be placed on the shared 2050-2069 window rather than each using its own longest usable period. The window is already in the cache name, so the two periods coexist.

## Data-quality rules the code enforces

- E3SMv3 daily max/min temperature are byte-identical to the daily mean at the source, so E3SM
  is excluded from temperature extremes. `MODELS_FOR()` keys on the variable, so precipitation
  and `tas`-based indices keep all four models automatically.
  
- `WSDI`, `CSDI` and `GSL` are annual-only by construction, since spells and seasons cross
  month boundaries. The code refuses them at `MS` and `QS-DEC`.

- Percentile thresholds are identical across scenarios, so counts are comparable
  between SSP2-4.5, G6-1.5K-SAI, and G6-1.5K-HiLLA.

- `_complete_bins` drops partial seasons before averaging, because xclim returns 0 rather than
  NAN for a partial bin and a truncated first or last winter would otherwise be counted as real
  low value.


## Known limitation (not fixed here)

`run_comparison_cached` builds its cache filename from model, index and comparison type, with no assessment window. That is safe while every call site passes the same window, which is currently true, but two windows would collide. Adding the window would rename existing caches and force a recompute, so it is left with a comment on the line.

## Version

`xclim==0.54.0` is pinned in `requirements.txt`. Season-length partial-bin behaviour is version dependent and `GSL` was validated on that version only.