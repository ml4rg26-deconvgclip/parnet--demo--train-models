# Demo — Fine-tuning PARNET on spliceosome data (HepG2)

This demo shows end-to-end fine-tuning of a pretrained PARNET model on a 9-RBP spliceosome
subset of the ENCODE eCLIP panel (HepG2 cell line). Starting from a pretrained 7M- or 21M-parameter
backbone, only the prediction head is replaced and trained — the full workflow runs in under
30 minutes on a single GPU for head-only fine-tuning.

**Selected RBPs**: AQR, BUD13, EFTUD2, PRPF8, RBM22, SF3B4, SMNDC1, U2AF1, U2AF2

---

## Quick run sequence

```bash
# ── Step 0 (one-off): preprocess and normalise the source dataset ─────────────
# See docs/preprocess-source-parnet-datasets.md for the full pipeline.
# Subsequent steps expect encode.filtered.pt under 600nt_windows.no-one-hot.stripped/.

# ── Step 1: filter source data to spliceosome RBPs ───────────────────────────
# Open and run:
#   notebooks/demo--train-spliceosome-hepg2/prepare_datasets.py.ipynb
#   → results/spliceosome-hepg2/datasets/dataset.pt  +  dataset.metadata.yaml
#   → results/spliceosome-hepg2/datasets/rbp_cts.tsv  +  tiles.bed

# ── Step 1b (optional): standalone BED6 export ───────────────────────────────
pixi run -e parnet-dev-cu11 python scripts/export_bed.py \
    --input  results/spliceosome-hepg2/datasets/dataset.pt \
    --format pt \
    --output results/spliceosome-hepg2/datasets/tiles.bed

# ── Step 2: fine-tune ────────────────────────────────────────────────────────
# Open and run:  notebooks/demo--train-spliceosome-hepg2/train_from_pretrained.py.ipynb
# Set params_name_of_resulting_model (e.g. "parnet.7m-0.0.ft-head.spliceosome-hepg2")
# → results/spliceosome-hepg2/training/<run_id>/model.statedict.pt

# ── Step 3: evaluate ─────────────────────────────────────────────────────────
# Open and run:  notebooks/demo--train-spliceosome-hepg2/evaluate_retrained_models.py.ipynb
# Set params_run_id = <run_id from Step 2>
```

---

## Prerequisites

- **pixi environment** — activate one of `parnet-dev-local`, `parnet-dev-cu11`, or `parnet-dev-cu12`.
  See the repo README for environment setup and the `externals/` symlinks.
- **Pretrained model weights** — `resources/models/parnet.7m-0.0.pt` (29 MB) must be present.
  Copy from NAS or unpack `resources/models.tar.gz`.
- **Source data** — the stripped 600 nt dataset (produced by Step 0):
  - `resources/parnet-encore-eclip/600nt_windows.no-one-hot.stripped/encode.filtered.pt` — canonical
    local source; native-length sequences with `pad_side` in meta; `GzListDataset(length=600)` re-pads
    at batch time.
- **GPU** — any CUDA-capable GPU with ≥8 GB VRAM for head-only fine-tuning; ≥16 GB for full fine-tuning.

All paths are configured in [`config/filepaths.yaml`](../config/filepaths.yaml).

---

## Notebooks — run in order

All notebooks live in
[`notebooks/demo--train-spliceosome-hepg2/`](../notebooks/demo--train-spliceosome-hepg2/).

---

### Step 0 (one-off) — Preprocess and normalise the source dataset

Run the full preprocessing pipeline once per machine. Full instructions, format
comparisons, and a metadata sidecar reference are in
[`docs/preprocess-source-parnet-datasets.md`](preprocess-source-parnet-datasets.md).

The subsequent steps expect:

- `resources/parnet-encore-eclip/600nt_windows.no-one-hot.stripped/encode.filtered.pt` — stripped
  native-length `.pt` (canonical source; sequences stored at genomic length, `pad_side` in meta).

Prepadded and HFDS variants are also produced by the pipeline and listed in
`config/filepaths.yaml` under `parnet_encore_eclip`; they are available as alternatives but not
required for this demo.

---

### Step 1 — Prepare the filtered dataset

**Notebook**: `prepare_datasets.py.ipynb`

Loads the stripped `encode.filtered.pt` (mmap, low RAM), filters to the 9 spliceosome RBPs and
to the demo chromosome subset, and writes:

```
results/spliceosome-hepg2/datasets/
    dataset.pt            ← dict with train/valid/test splits (ready for GzListDataset)
    dataset.metadata.yaml ← metadata sidecar (total_key, n_tracks, seq_len, splits, …)
    rbp_cts.tsv           ← RBP name → track index in the full 223-track dataset
    tiles.bed             ← BED6 with per-tile genomic coordinates and pad_side
```

Key parameters:

| Parameter | Description | Default |
|---|---|---|
| `PARAMS_RBP_SET` | Set of RBP names to include | 9 spliceosome RBPs |
| `PARAMS_CELL_LINE` | Cell line filter applied to `full_rbp_set.tsv` | `"HepG2"` |
| `PARAMS_MAX_CHROMS_PER_SPLIT` | Demo subset size (chromosomes per split) | `3` |
| `PARAMS_MIN_READ_COUNT` | Minimum eCLIP read count to retain a tile | `3` |

Expected runtime: ~5–10 min (disk-bound, not GPU).

**HFDS output (optional)**: the last cell of the prepare notebook documents the command to
convert the output `.pt` to HFDS format using `scripts/convert_pt_to_hfds.py`:

```bash
pixi run -e parnet-dev-cu11 python scripts/convert_pt_to_hfds.py \
    --input     results/spliceosome-hepg2/datasets/dataset.pt \
    --outputdir results/spliceosome-hepg2/datasets/dataset.hfds \
    --total-key eCLIP
```

---

### Step 2 — Fine-tune the model

**Notebook**: `train_from_pretrained.py.ipynb`

Key parameters in the `### Parameters` cell:

| Parameter | Description | Default |
|---|---|---|
| `params_dataset_format` | `"pt"` (Step 1 output) or `"hfds"` (optional HFDS variant) | `"pt"` |
| `params_seq_length` | Sequence length passed to `GzListDataset` / `HFDSDataset` | `600` |
| `params_name_of_resulting_model` | Output subfolder name under `results/…/training/` | `"parnet.7m-0.0.ft-head.spliceosome-hepg2"` |
| `params_head_variant` | `"additive_mix_max"` recommended (matches 7M/21M pretraining) | `"additive_mix_max"` |
| `params_finetuning_strategy` | `"head"` (fast) / `"unfreeze_last_n_layers"` / `"full"` | `"head"` |
| `params_penalty_factor` | Mixing-coefficient regularisation (0 = none, 5 = moderate) | `5.0` |
| `params_max_epochs` | Training epochs | `5` |
| `params_batch_size` | Batch size | `32` |

**Output directory**: `results/spliceosome-hepg2/training/{params_name_of_resulting_model}/`

Contents after training:

```
checkpoints/best.ckpt, last.ckpt
csv_logs/version_0/metrics.csv
tensorboard_logs/version_0/
model.statedict.pt     ← lightweight bundle (load via load_parnet_model_from_statedict)
model.full.pt          ← full model object (torch.save/load)
run_config.yaml        ← all training hyperparameters (read by evaluate notebook)
```

Expected runtime:
- `"head"` strategy: ~5–30 min (9 tasks, ~22 k train tiles)
- `"full"` strategy: several hours

**Monitoring**: TensorBoard is integrated (Option C cell in the Training section).
The in-memory `MetricHistory` callback (Option B) produces quick matplotlib plots
immediately after `trainer.fit()`.

---

### Step 3 — Evaluate the fine-tuned model

**Notebook**: `evaluate_retrained_models.py.ipynb`

Set `params_run_id` to the value of `params_name_of_resulting_model` used in Step 2.
All other parameters (`params_seq_length`, `params_num_tasks`, `params_dataset_format`)
are read automatically from `run_config.yaml`.

Key parameters:

| Parameter | Description | Default |
|---|---|---|
| `params_run_id` | Run folder name (= `params_name_of_resulting_model` from Step 2) | see notebook |
| `params_dataset_format` | Must match Step 2 (`"pt"` or `"hfds"`) | `"pt"` |
| `params_test_dataset_path_override` | Optional path to an alternative test dataset; format is auto-detected (directory → `HFDSDataset`, file → `GzListDataset`). Set to `None` to use the default path. | `None` |

**Outputs**:

| Section | What you get |
|---|---|
| Training metrics | Loss and mix-coeff curves from `metrics.csv` |
| Test-set evaluation | Per-RBP Pearson r and Spearman rho table + seaborn boxplots |
| Profile comparison | 3-row grid: ground truth / fine-tuned / pretrained predictions |

The profile comparison panel shows the test sequence with the strongest combined signal
for `params_show_rbp_names` (default: PRPF8, U2AF2, AQR).

---

## Dataset classes

The train and evaluate notebooks use patched dataset classes from
`parnet_additional_utils.patch_datasets`:

| Class | Source | Use |
|---|---|---|
| `GzListDataset` | `.pt` or `.pt.gz` file (Step 1 output) | Streaming with optional mmap companion `.pt`; re-pads native-length sequences at batch time |
| `HFDSDataset` | HuggingFace `DatasetDict` (optional HFDS output) | Streaming from Arrow shards; re-pads at batch time |
| `PreloadedListDataset` | any list of dicts | Pre-loads all samples into RAM |

All three return batches matching the LightningModel contract:
`batch["inputs"]["sequence"]` (4, L) and `batch["outputs"]["total"]` (T, L).

`GzListDataset` and `HFDSDataset` receive `total_key="eCLIP"`, `control_key="control"`, and
`length=600` explicitly in the notebooks — these are not inferred from the metadata sidecar.

---

## Configuration reference

| File | Purpose |
|---|---|
| `config/filepaths.yaml` | All data paths (models, datasets, metadata) |
| `config/config.train_validation_test_split.yaml` | Chromosome assignments for train/valid/test |
| `results/…/run_config.yaml` | Per-run snapshot of all training hyperparameters |

---

## Outdated: the 2000 nt pt.gz and prepadded 600 nt workflows

Earlier versions of this demo used a 2000 nt `.pt.gz` source or the prepadded 600 nt HFDS
(sequences stored at fixed 600 nt with N-padding). Both required extra prepare steps:
the 2000 nt dataset needed center-cropping (`CROP_START=700, CROP_END=1300`); the prepadded
HFDS lacked per-tile `pad_side` metadata and had sequences stored as one-hot tensors.

The legacy prepare code is preserved for reference in:
`notebooks/old_ref/prepare_datasets.legacy.from-2000nt-or-600nt-prepadded-pt.py.ipynb`

The stripped 600 nt dataset (this demo's canonical source) is equivalent in content,
loads in seconds via mmap, and carries correct `pad_side` metadata so no manual cropping
is needed.
