# morphology-omnipose

Notebooks for segmenting bacterial cells from phase-contrast images with
[Omnipose](https://github.com/kevinjohncutler/omnipose), QC'ing the results, and flagging
morphological outliers (e.g. long/tangled cells) — as a replacement for an older Cellpose-based
segmentation pipeline. Includes tools to compare the two pipelines side by side.

All notebooks run in the `omnipose` conda environment:

```
conda activate omnipose
jupyter notebook
```

## Pipeline

1. **`omnipose_segment.ipynb`** — runs the `bact_phase_omni` model on `.phase.tif` images.
   Cells 1–7 run locally on a single FOV (useful for testing, may run out of memory on large
   images). The final cells submit one LSF (`bsub`) job per image on WEXAC instead.
   Saves per-FOV outputs (`seg.npy`, `seg.unfilt.npy`, `props.txt`, `phase_per_cell.txt`,
   `seg_image.tif`) into `omnipose_seg/{fov_name}/`.

2. **`run_omnipose_seg_wexac.ipynb`** — a cleaner, WEXAC-only version of the batch step above.
   Submits one `bsub` job per phase image via `lsf_omnipose_seg_morph.py`, and includes cells to
   monitor job status (`bjobs`), check which FOVs finished, and inspect failed-job logs.

3. **`filter_long_cells.ipynb`** / **`filter_long_cells_by_length.ipynb`** — flag morphological
   outliers per FOV: cells whose `area` (resp. `major_axis_length`) exceeds a threshold × the
   per-FOV mean. Saves a long-cell mask + detail table per FOV, plots the distribution with the
   threshold overlaid (for tuning), and includes a napari cell to inspect flagged cells in
   context.

4. **`omnipose_seg_qc.ipynb`** — manual QC. Opens napari for each FOV with the phase image, the
   current Omnipose mask, the legacy Cellpose result as a green/red reference (kept vs.
   manually-dropped cells), and long-cell flags (optional). Paint over cells to remove them;
   closing the window saves the filtered mask (`seg.npy`) and the removed cells
   (`seg.manual_qc.npy`), then automatically advances to the next pending FOV.

5. **`compare_seg.ipynb`** — side-by-side visual comparison of Cellpose vs. Omnipose for a single
   FOV: additive-blended binary masks in napari (green = Cellpose post-QC, magenta = Omnipose,
   cyan = Omnipose long-cell flags).

6. **`view_segmentation.ipynb`** — lightweight viewer for saved Omnipose results (matplotlib
   overlay + optional napari), no QC/editing.

7. **`roy_segmentation_qc.ipynb`** — exploratory notebook from a related QC pipeline
   (`seg_qc_functions`). Covers autofluorescence/sediment signal cleanup, an alternative manual
   napari QC workflow, filtering cells with bad demultiplexing signatures, and training an
   XGBoost classifier on per-cell morphological features (area, eccentricity, solidity, etc.) to
   predict which cells a human would keep — an automated alternative to manual QC.

## Data layout

Notebooks expect an experiment folder with these subfolders (names/paths are set in each
notebook's settings cell):

```
{experiment}/
├── phase-proj/                     # {fov}_hyb_{h}.phase.tif input images
├── old_seg/                        # legacy Cellpose results (reference only)
│   └── fov_N_hyb_H/
│       ├── fov_N_hyb_H.seg.npy             # kept cells
│       └── fov_N_hyb_H.seg.manual_qc.npy   # manually dropped cells
├── omnipose_seg/                   # Omnipose results (this pipeline's output)
│   └── fov_N_hyb_H/
│       ├── fov_N_hyb_H.seg.npy             # working/filtered mask
│       ├── fov_N_hyb_H.seg.unfilt.npy      # raw model output
│       ├── fov_N_hyb_H.seg.manual_qc.npy   # cells removed during QC
│       ├── fov_N_hyb_H.props.txt           # per-cell morphology
│       ├── fov_N_hyb_H.phase_per_cell.txt  # per-cell phase intensity stats
│       └── fov_N_hyb_H.seg_image.tif
└── long_cell_filter/ or long_cell_filter_by_length/
    └── fov_N_hyb_H/
        └── fov_N_hyb_H.seg.long_cells.npy
```

Local paths point at a Box-synced folder (`~/Box/Zohar_Persky/projects/p2f-revisions/morph-omnipose`);
cluster notebooks point at the equivalent path on WEXAC. Edit the `_base`/`EXP_DIR` settings cell
at the top of each notebook to switch experiments or environments.

## Running segmentation on WEXAC

`run_omnipose_seg_wexac.ipynb` (and the WEXAC cell in `omnipose_segment.ipynb`) submit one LSF job
per FOV via `bsub`, running `lsf_omnipose_seg_morph.py` in the `omnipose` conda env. Use the WEXAC
JupyterHub (`https://jupyter.wexac.weizmann.ac.il`) or SSH in directly. Once all jobs show `DONE`
in `bjobs`, run `omnipose_seg_qc.ipynb` locally to review results.
