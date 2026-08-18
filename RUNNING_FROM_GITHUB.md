# Computing an ETCCDI index from this repository

<<<<<<< HEAD
Anyone with access to the Reflective Cloud Hub can compute any of the twelve
percentile-based and fixed threshold indices without writing code, using `compute_any_index.ipynb`.
=======
Anyone with access to the Reflective Cloud Hub can compute any of the eight
percentile indices and without writing code, using `compute_any_index.ipynb`.
>>>>>>> 4fbce5c (Add fixed-threshold, seasonal, and monthly indices)

## 1. Clone the repository on the hub

Open a terminal in JupyterLab and run:

    git clone https://github.com/ReflectiveCloud/etccdi-indices.git
    cd etccdi-indices

(While the percentile work is still on a branch, add `-b francis-percentile-indices`
to the clone command, or `git checkout francis-percentile-indices` afterwards.)

## 2. Open the notebook

In the JupyterLab file browser, navigate into `etccdi-indices/` and open
`compute_any_index.ipynb`. Choose the standard pangeo Python 3 kernel.

The notebook imports `multimodel_etccdi.py`, which sits beside it in the same
directory, so the import resolves with no path setup. Nothing to install: the
hub image already carries xclim, xarray, s3fs, and the rest.

## 3. Edit one cell and run

Only the USER CHOICES cell needs editing:

    INDEX     = 'TX90p'     # percentile: R95p R99p TX90p TX10p TN90p TN10p WSDI CSDI
                            # fixed-threshold: RX1D RX5D GSL GDD
    MODELS    = ['CESM']    # 'CESM', 'UKESM', 'MIROC', 'E3SM'
    SCENARIO  = 'HiLLA'     # 'SSP245', 'SAI', 'HiLLA'
    MEMBERS   = None        # None = all available
<<<<<<< HEAD
    FREQ      = 'YS'        # 'YS' annual (default), 'MS' monthly, 'QS-DEC' (seasonal cycle)
=======
    FREQ      = 'YS'        # 'YS' annual, 'MS' monthly, 'QS-DEC' DJF season

    MAKE_PLOT = True        # quick-look map at the end
    SAVE_PLOTS= False       # True writes each figure as PNG
>>>>>>> 4fbce5c (Add fixed-threshold, seasonal, and monthly indices)

Then Run All. The notebook prints what it is loading, computes the index, and
plots the ensemble mean.

## 4. What you get back

    results[model]['per_member']   # {member: DataArray}
    results[model]['ens_mean']     # ensemble mean

At `FREQ='YS'` these are `(lat, lon)`: mean days per year.
At `FREQ='MS'` they are `(month, lat, lon)`: mean days per calendar month, the
seasonal cycle. Summing over `month` recovers the annual field.
At `FREQ='QS-DEC'` they are `(lat, lon)` for DJF only.

Each field carries `units`, `long_name` and a `comment` describing how it was computed, so a file written to disk explains itself.

With `MAKE_PLOT = True` you also get a map with coastlines and national borders and for monthly runs a twelve-panel climatology plus the cosine-latitude weighted seasonal cycle. `SAVE_PLOTS = True` writes each figure as a PNG named for the index, model, scenario and window

## 5. Cost, before you pick a big configuration

The expensive step is the baseline percentile threshold, computed once per
model per variable and cached for the session. A single model with all members
takes roughly 30-40 minutes on the hub for a percentile index; four models scale accordingly. Memory grows with the member count, since each member holds its daily record. Keep the
kernel alive between runs on the same model and variable: the second index is
much faster because the threshold is reused.

The four fixed-threshold indices (`RX1D`, `RX5D`, `GSL`, `GDD`) skip the percentile step entirely and finish faster. Monthly and DJF frequency cost no more than annual, since the threshold is shared.

E3SM's first access to a variable triggers cubed-sphere regridding and caching
to the bucket, which is slow once and fast afterwards.

## 6. Rules the code enforces to have it running smoothly

- E3SM is skipped for temperature indices built on daily temperature extremes (`TX90p`,
  `TX10p`, `TN90p`, `TN10p`, `WSDI`, `CSDI`): its archived daily max and min are identical
  to the daily mean at source, so no true daily extremes exist. Precipitation indices and the
  `tas`-based `GSL` and `GDD` keep all four models.
  
- `WSDI`, `CSDI` and `GSL` are refused at `FREQ='MS'` and `FREQ='QS-DEC'`: their spells and
  seasons cross month boundaries, so a sub-annual count is not the ETCCDI quantity.

- Percentile thresholds always come from SSP2-4.5 2020-2039, whatever scenario
  you compute, so counts are comparable across SSP2-4.5, G6-1.5K-SAI, and
  G6-1.5K-HiLLA.

- `GSL` uses a hemisphere-aware wrapper, since a southern-hemisphere growing season straddles
  the calendar year: northern gridpoints use a 1 July mid-date on a calendar year, southern
  gridpoints a 1 January mid-date on a July-to-June year. Without this, southern values are
  pinned by the date convention rather than by temperature

## 7. Sharing a result

Set `SAVE_TO_BUCKET = True` to write per-member fields into the shared S3
layout via the pipeline's own writer:

    .../ETCCDI_indices_annual/{model}/{scenario}/{member}/{INDEX}.nc     # YS
    .../ETCCDI_indices_monthly/{model}/{scenario}/{member}/{INDEX}.nc    # MS

Only do this for configurations worth sharing.

## 8. Adding your own indicator

`INDEX_REGISTRY` in `multimodel_etccdi.py` maps an index name to its variable,
its percentile (or `None` for a fixed threshold), the xclim function and optional `attrs` supplying `long_name` and `comment`. Add an entry and the notebook picks it up with
no other change: it appears in the printed index list and every cell works as
before. 

`MODELS_FOR()` keys on the variable, so an index on `pr` or `tas` automatically keeps all four models and one on `tasmax` or `tasmin` automatically drops E3SM. If the index is spell-based, add it to `SPELL_INDICES` so the frequency guard applies

If your indicator needs a variable the loaders do not carry yet (wind,
humidity), that is a loader addition; open an issue or a pull request.

## 9. Version

The pipeline pins `xclim==0.54.0`. Season-length partial-bin behaviour is version dependent and `GSL` was validated on that version only. If the hub image moves ahead, re-run the southern hemisphere check in the repository before trusting `GSL` output.