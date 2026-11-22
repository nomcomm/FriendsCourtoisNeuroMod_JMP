# Friends fMRI — Reproducible Analysis & Figure Package

Analysis code for: Schmälzle, R., & Sondhi, M. (2026). The One Where The Brains Sync:
Audience Synchronization and Laughter-Associated Modulation Across Six Seasons of
*Friends*. *Journal of Media Psychology*. https://doi.org/10.1027/1864-1105/a000529

Self-contained package that **reproduces every result and figure** in the paper. The
analysis and figures run from the files in this repository alone; the raw source data
(CNeuroMod BOLD, stimulus audio) is not included and only matters for an optional
from-raw re-derivation (see below).

## Principle

The folder tree tells the story. Execution order comes from the block prefix
(`0_ → a_ → b_ → c_`), and every script is named for **what it does**. `data/` mirrors
`scripts/` exactly: a script in `scripts/<block>/` writes only to `data/<block>/`.

## Structure

```
config.py            all paths & parameters, in one place
run_all.py           the orchestrator — defines & runs the pipeline in order
scripts/
  0_prep/            foundations (timeseries, classifier, annotations)
  a_isc_stability/   BLOCK A — stable shared ISC at scale (Part A)
  b_laughter/        BLOCK B — laughter modulation: GLM (primary) + ISC + content
  c_controls/        BLOCK C — control analyses (supplement)
data/                mirrors scripts/ — carried inputs + derived outputs
assets/              bundled reference atlases (MNI template, Melbourne subcortex)
results/
  tables/
  analysis_plots/    interim analysis plots the pipeline emits (quick-look + source panels)
figures/             THE paper figures — assembles panels into figure1–3 + figS1–8
  scripts/ panels/ output/ figure_plan.md
```

Note the two distinct roles: `results/analysis_plots/` are pipeline byproducts; `figures/`
is where the **paper** figures are made (it stages a curated subset of the analysis plots
and composes the final figures). The blocks map 1:1 to the parts of
the paper (A: ISC stability, B: laughter modulation, C: control analyses).

## How to run

```bash
python run_all.py                 # reproduce all results (regenerates data/ + analysis_plots/)
python run_all.py --list          # print the pipeline stages
bash figures/scripts/build_all.sh # assemble the paper figures into figures/output/
```

The cheap analysis stages are deterministic: a `--force` re-run reproduces every number
**bit-for-bit** (verified 2026-07-31 in an isolated clean-room copy; max|Δ| = 0 across all
key outputs).

## Conventions

- Each script is config-driven (`config.py`) and self-contained; no hardcoded paths.
- Scripts **skip if their output exists** (safe to re-run); most also accept `--force`,
  and the plotting/summary stages accept `--plot-only` (re-render from saved outputs).
- Primary pipeline = **fMRIPrep**; the original H5 pipeline is corroborating evidence only.
- Primary laughter classifier = **Clf-C** (86-feature RF, human labels s01e01a + s04e09a; κ ≈ 0.6).

## What's carried vs. what regenerates

The expensive raw stages were run once; their outputs are **carried** in `data/` (each
subfolder has a `PROVENANCE.txt`). A normal run therefore reproduces the analysis+figures
from those carried intermediates:

| stage | cost | needs |
|---|---|---|
| `a_*`, `b_*` (parcel GLM/ISC/striatum/dose/season/av-energy/humor-type), `c_*` controls | **seconds–minutes** | only carried inputs — cheap, deterministic |
| `0_prep/extract_timeseries` | ~1.5 TB, hours | raw BOLD from the CONP mirror, not included (timeseries carried → normal run skips) |
| `0_prep/predict_laughter` | minutes | stimulus audio, not included (model + annotations carried → skips) |
| `humor_classification` | LLM API | Gemini key (per-TR labels carried → skips) |
| `glm_contrast_voxelwise.py` | hours, ~1.5 TB read | raw BOLD, not included; **not in `run_all`** — heavy voxelwise step. Its *result* (`data/b_laughter/glm_voxelwise/group_tmap*.nii.gz`) is carried here and feeds Figure S1; only the per-voxel `betas`/`betas_motion` are external. |

## Raw sources (not included)

The raw source data (CNeuroMod fMRIPrep BOLD and confounds, the stimulus audio/video, and
the files of the original submission) is too large to distribute here and is kept on
external storage. `config.py` points `ORIGINAL_DIR` / `CNEUROMOD_DIR` / `CONFOUNDS_DIR`
(and the stimulus/BIDS roots) to that storage; set these paths to your own copy of the
CNeuroMod *Friends* dataset for a from-raw re-derivation. **The analysis and figures never
touch them** — they run from the bundled `assets/` and the carried `data/` (e.g.,
`head_motion.py --force` reads the raw confounds; `head_motion.py --plot-only` reproduces
the control from the carried result without them).

## Inherited numbers (not regenerated here)

`scripts/verify_claims.py` runs last and prints every manuscript number the package
produces. It closes with an explicit **INHERITED** list: published values whose
source code is the original submission's notebooks, which are not in this tree.
They are reported as published and are *not* recomputed, because recomputing them
with this package's methods gets close but not equal — substituting those numbers
would silently contradict the printed supplement. The list covers the H5
laughter-ISC ROI table (and its 474/1000, and the r = 0.961 pipeline agreement),
the legacy-classifier event/TR comparison, the boundary and alternative-detector
κ values, the humor schema-adherence figures, and Figure S3.

H5 *spatial stability* (0.867 ± 0.030) was on that list until
`c_controls/h5_corroboration.py` was added; it reproduces exactly from the carried
`isc_h5/` maps and is now printed as block **C5**.

## Notes / latent items

- Rebuilding `data/` fully from raw is **not** one command (the carry-over was done once,
  per-folder `PROVENANCE.txt`); the reproducible unit is analysis + figures from carried data.
- **AppleDouble hazard:** an exFAT external drive accrues `._` files; any script that
  `pathlib.glob("*.npy")` over such a path must skip names starting with `._`
  (already handled in `glm_contrast_voxelwise.py`).
- `config.EXCLUDED_EPISODES` vs. per-episode coverage: a mismatch was flagged in
  `qc_coverage.py` and left as-is (does not affect the reported analyses).
