# Demo — Fine-tuning PARNET on the full filtered eCLIP panel

This demo fine-tunes a pretrained PARNET model on the **full 223-RBP ENCODE eCLIP panel**,
pre-filtered to retain tiles with at least one RBP showing meaningful total CLIP signal.
Unlike the spliceosome demo there is no dataset-preparation step — the filtered dataset
lives on NAS and is referenced directly via `config/filepaths.yaml`.

---

## Prerequisites

- **pixi environment** — activate one of `parnet-dev-local`, `parnet-dev-cu11`, or `parnet-dev-cu12`.
  See the repo README for environment setup.

- **Pretrained model weights** — `resources/models/parnet.7m-0.0.pt` or
  `resources/models/parnet.21m-0.0.pt`. Copy from NAS or unpack `resources/models.tar.gz`.

- **Filtered dataset** — NAS path configured in `config/filepaths.yaml`:

  ```yaml
  results:
    filtered_dataset_full_set_total_clip: /mnt/storage-nas-fast-2/…/eclip_600bp_signalfiltered.pt.gz
  ```

  The NAS must be mounted. No local conversion step is needed.

- **GPU** — ≥8 GB VRAM for head-only fine-tuning; ≥16 GB for full fine-tuning.

---

## Quick run sequence

```bash
# ── Step 1: fine-tune (223 RBPs, full filtered dataset) ──────────────────────
# Open and run:
#   notebooks/demo--train-full-set-filtered-for-total-clip/train_from_pretrained.py.ipynb
# Key parameters:
#   params_dataset_format  = "ptgz"   (the NAS .pt.gz)
#   params_name_of_resulting_model  e.g. "parnet.7m-0.0.ft-head.full-set-filtered"
# → results/spliceosome-hepg2-ptgz/training/<run_id>/model.statedict.pt

# ── Step 2: evaluate ─────────────────────────────────────────────────────────
# Open and run:
#   notebooks/demo--train-full-set-filtered-for-total-clip/evaluate_retrained_models.py.ipynb
# Set params_run_id = <run_id from Step 1>

# ── (optional) dataset and prediction exploration ────────────────────────────
# Open:  notebooks/demo--train-full-set-filtered-for-total-clip/demo_explore_predictions_and_datasets.py.ipynb
# Validates dataset consistency across pt.gz / HFDS / BigWig / FASTA, and benchmarks
# the pretrained 7M model on 1,000 test tiles.
```

---

## Notebooks

All three notebooks live in
[`notebooks/demo--train-full-set-filtered-for-total-clip/`](../notebooks/demo--train-full-set-filtered-for-total-clip/).

| Notebook | Purpose |
|---|---|
| `train_from_pretrained.py.ipynb` | Fine-tune from pretrained weights; saves model + run config |
| `evaluate_retrained_models.py.ipynb` | Per-RBP Pearson/Spearman correlations, boxplots, profile comparison |
| `demo_explore_predictions_and_datasets.py.ipynb` | Dataset consistency checks; pretrained model baseline |

---

## Key parameters (`train_from_pretrained.py.ipynb`)

| Parameter | Description | Default |
|---|---|---|
| `params_dataset_format` | `"ptgz"` for the NAS `.pt.gz` | `"ptgz"` |
| `params_name_of_resulting_model` | Output folder name under `results/…/training/` | see notebook |
| `params_head_variant` | `"additive_mix_max"` recommended (matches pretraining) | `"additive_mix_max"` |
| `params_finetuning_strategy` | `"head"` / `"unfreeze_last_n_layers"` / `"full"` | `"head"` |
| `params_penalty_factor` | Mixing-coefficient regularisation | `5.0` |
| `params_max_epochs` | Training epochs | `5` |

---

## Configuration reference

| File | Purpose |
|---|---|
| `config/filepaths.yaml → results.filtered_dataset_full_set_total_clip` | NAS path to the pre-filtered dataset |
| `config/filepaths.yaml → models` | Pretrained weight paths |
| `results/…/run_config.yaml` | Per-run snapshot of training hyperparameters |
