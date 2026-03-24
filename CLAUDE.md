# CLAUDE.md — AI Assistant Guide for plateanalysis_Rapha

## Project Overview

This is a scientific pipeline for detecting **transient and variable astronomical objects** in archival photographic plate data from the [APPLAUSE archive](https://www.plate-archive.org/applause/). The workflow compares consecutive photographic plate exposures of the same sky region to identify sources that appear in one plate but not the other — candidates for short-timescale transients (e.g., novae, flares, moving objects).

The project is **Jupyter notebook-driven** with shared utility code in `footprints/library.py` and configuration in `footprints/settings.py`.

---

## Repository Structure

```
plateanalysis_Rapha/
├── README.md
├── CLAUDE.md                          # This file
└── footprints/                        # All project code lives here
    ├── library.py                     # Core functions and Worker classes (~1400 lines)
    ├── settings.py                    # Configuration, parameters, and sequence definitions
    ├── dataset.json                   # Tracks current active plate pair and sequence
    ├── images.json                    # Maps plate IDs → FITS filenames (119 entries)
    │
    ├── download.ipynb                 # Step 1: Download FITS images and catalogs
    ├── footprint_analysis.ipynb       # Step 0: Find suitable plate sequences
    ├── find_mismatches.ipynb          # Step 2: Match sources between plate pairs
    ├── psf_analysis.ipynb             # Step 3: PSF fitting and shape analysis
    ├── display_nonmatches.ipynb       # Step 4: Visualize and vet candidates
    ├── display_sequence.ipynb         # Utility: Display full sequences
    ├── pipeline.ipynb                 # Batch orchestration across sequences
    ├── pipeline_results.ipynb         # Batch result summaries
    ├── event_analysis.ipynb           # Advanced event analysis
    ├── collate.ipynb                  # Collate results across sequences
    ├── results.ipynb                  # Results visualization
    ├── results_analysis.ipynb         # Statistical analysis of results
    │
    └── results/                       # Output data
        ├── best.csv                   # Human-readable candidate list
        ├── candidates_all.fits        # All candidates (FITS table)
        └── candidates_best.fits       # Filtered best candidates (FITS table)
```

---

## Analysis Pipeline (Execution Order)

1. **`footprint_analysis.ipynb`** — Query APPLAUSE catalog; identify plate pairs with >50% field overlap from the same night; build `sequences` dict in `settings.py`
2. **`download.ipynb`** — Fetch FITS images and SExtractor source catalogs from APPLAUSE; update `images.json`
3. **`find_mismatches.ipynb`** — Cross-match source catalogs between plate pairs using multiprocessing `Worker` class; output matched/non-matched FITS tables
4. **`psf_analysis.ipynb`** — Fit 2D Gaussians and extract radial profiles for non-matched sources using `FitWorker` and `ProfileWorker`; apply shape criteria
5. **`display_nonmatches.ipynb`** — Side-by-side image cutout visualization; apply `exceeds_criteria()` filters; manual review step
6. **`collate.ipynb`** / **`pipeline_results.ipynb`** — Aggregate candidates from all sequences; export final FITS/CSV results
7. **`pipeline.ipynb`** — Orchestrates steps 3–5 in batch for all sequences

---

## Key Files In Depth

### `footprints/settings.py`

Central configuration. Key elements:

- **`DATAPATH`**, **`CATALOG`**, **`RESULTS`** — filesystem paths
- **`fname(plate_id)`** — construct data file paths
- **`get_table_sources()`**, **`get_table_psf_nomatch()`**, **`get_table_matched()`** — generate standard output filenames
- **`get_parameters(dataset_id)`** — returns parameter dict for a plate pair (falls back to `'default'`)
- **`parameters`** dict — all analysis thresholds; `parameters['default']` is the baseline, per-pair keys (e.g., `'9341,9342'`) override specific values
- **`sequences`** dict — maps `'seq01'` through `'seq40'` to ordered lists of plate ID strings

**When adding a new plate sequence**: add an entry to `sequences` in `settings.py` and add per-pair parameter overrides if telescope characteristics differ.

### `footprints/library.py`

All reusable functions and multiprocessing classes. Key sections:

| Function/Class | Purpose |
|---|---|
| `update_dataset()`, `update_sequence()` | Read/write `dataset.json` |
| `is_false_positive()` | Aperture-photometry flux ratio check |
| `exceeds_criteria()` | Apply multi-threshold filters (profile diff, elongation, circularity, flux ratio) |
| `fit_fwhm()` | 2D Gaussian PSF fitting (wraps photutils) |
| `make_radial_profile()` | Radial brightness profile extraction |
| `clean_bad_fits()` | Remove poorly-fitted sources |
| `get_cutouts()` | Extract image stamps around sources |
| `get_pixel_coords()` | WCS → pixel coordinate conversion |
| `plot_psf_analysis()` | Multi-panel PSF comparison figures |
| `plot_analysis_results()` | Full analysis visualization |
| `get_earth_shadow()` | Earth shadow sky projection |
| **`Worker`** | Multiprocessing coordinate-matching class |
| **`FitWorker`** | Parallel 2D Gaussian PSF fitting |
| **`ProfileWorker`** | Parallel radial profile + circularity analysis (uses OpenCV) |

### `footprints/dataset.json`

Tracks the **currently active** plate pair and sequence. Updated by notebooks as processing advances:

```json
{
    "current_dataset": "9319,9320",
    "current_sequence": "seq03"
}
```

### `footprints/images.json`

Maps integer plate IDs (as strings) to FITS filenames:

```json
{
    "9319": "GS00768_x.fits",
    "9320": "GS00769_x.fits"
}
```

---

## Coding Conventions

### Naming
- **Plate identifiers**: plain integers (e.g., `9319`)
- **Plate pairs / dataset keys**: comma-separated string (e.g., `'9319,9320'`)
- **Sequences**: `'seq##'` strings (e.g., `'seq03'`)
- **Functions**: `snake_case` with action verbs (`get_`, `plot_`, `make_`, `fit_`, `extract_`, `clean_`)
- **Table column names**: follow SExtractor conventions (`x_fit`, `y_fit`, `flux`) plus ICRS coordinates (`ra_icrs`, `dec_icrs`)

### Data Structures
- **Astropy Tables** for all source catalogs (`.fits` files via `astropy.io.fits`)
- **FITS format** for all output tables and images
- **JSON** for lightweight configuration state
- **CSV** for human-readable candidate exports

### Multiprocessing Pattern
Worker classes are designed for `multiprocessing.Pool.apply_async()`:

```python
class SomeWorker:
    def __init__(self, ...):
        ...
    def __call__(self, args):
        # work here; catch exceptions internally
        try:
            ...
        except Exception as e:
            return None, str(e)
```

Always use `__call__()` so the instance is picklable for inter-process communication.

### Parameter Management
- All thresholds live in `settings.parameters`
- Use `get_parameters(dataset_id)` to retrieve the correct dict for any plate pair
- When tuning for a new telescope/dataset, add a key to `parameters` rather than hardcoding values in notebooks

### Error Handling
- Use try/except in Worker classes to prevent one bad source from crashing a batch
- Suppress known Astropy/ERFA deprecation warnings at the top of notebooks
- JSON reads should handle `FileNotFoundError` and `JSONDecodeError`

---

## Dependencies

No `requirements.txt` is present. Required packages:

| Package | Usage |
|---|---|
| `numpy` | Array operations |
| `astropy` | FITS I/O, WCS, sky coordinates, tables, time |
| `photutils` | PSF fitting, aperture photometry, radial profiles, background |
| `reproject` | Image reprojection between WCS frames |
| `opencv-python` (`cv2`) | Contour analysis, circularity/shape metrics |
| `matplotlib` | All plots and figures |
| `earthshadow` | Earth shadow sky projections |
| `erfa` | High-precision coordinate transforms (via astropy) |

Install with:
```bash
pip install numpy astropy photutils reproject opencv-python matplotlib earthshadow
```

---

## Development Workflow

### Git Branches
- `main` / `master` — stable releases
- `dev` — development branch; merge PRs here before promoting to main
- Feature branches: `claude/<description>` for AI-driven changes, or descriptive names for manual work

### Running the Pipeline
1. Set working directory to `footprints/`
2. Run notebooks in the order listed in the pipeline section above
3. `dataset.json` tracks progress; notebooks read/write this file to pass state between steps
4. Use `pipeline.ipynb` for automated batch runs over all sequences

### Adding a New Plate Sequence
1. Add sequence entry to `sequences` dict in `settings.py`:
   ```python
   sequences['seq41'] = ['12345', '12346', '12347']
   ```
2. Add corresponding plate-pair entries to `parameters` if telescope settings differ from defaults:
   ```python
   parameters['12345,12346'] = {
       'fwhm_init': 10,
       'fit_shape': 35,
       # only override what differs from 'default'
   }
   ```
3. Add plate ID → filename mappings to `images.json`
4. Run the pipeline from `find_mismatches.ipynb` forward

### Modifying Analysis Criteria
- Edit `parameters['default']` in `settings.py` for global changes
- Add per-pair overrides for telescope-specific tuning
- Key threshold parameters:
  - `profile_diff_threshold` — profile shape comparison cutoff
  - `circularity_threshold`, `circularity_low_limit` — shape roundness filters
  - `elongation` — source elongation limit (SExtractor flag)
  - `max_flux_threshold`, `min_acceptable_flux` — flux range filters
  - `fwhm_init`, `min_fwhm`, `max_fwhm` — PSF fitting range (pixels)

---

## Important Notes for AI Assistants

1. **Do not modify `dataset.json` or `images.json` unless explicitly asked** — these track live pipeline state
2. **All analysis parameters belong in `settings.py`**, never hardcoded in notebooks or `library.py`
3. **`library.py` functions must remain picklable** (no lambda captures, no unpicklable objects stored on Worker instances) for multiprocessing compatibility
4. **Notebooks are ordered steps**, not independent scripts — they share state via FITS files on disk and `dataset.json`
5. **FITS table column names** follow SExtractor conventions — do not rename columns unless updating all downstream consumers
6. **Results are science outputs** — changes to filtering thresholds should be discussed/documented, not silently adjusted
7. **No test suite exists** — validate changes by running affected notebooks end-to-end on a small sequence (e.g., seq03)
