# README - PARNET Demo - Training Datasets and train models

## Overview

Repo aimed at demonstrating details about how to train PARNET.

Currently focusing on using pretrained models to speed up the process + training on filtered data; see the "spliceosome-hepg2" demo notebooks.

## Set-up

### General organisation

The project assumes a multi-repo structure where a repository is dedicated to a specific section of the project,
somewhat isolated from the rest.

We also suggest to work through VSCode Remote SSH and its multi-root workspaces,
which allow to edit multiple repos at once and easily jump between them.

Clone this repo and its sibling dependencies into a shared parent folder:

```bash
projects/
├── project-parnet/                               ← The parent folder for all the repos you plan to edit.
│   ├── meta/
│   │   └── project-parnet-meta.code-workspace/   ← A suggested VSCode workspace file.
│   │       └── .git/
│   └── parnet--demo/                             ← Example of sub-section if you want to organize your repositories. Not a repo.
│       └── parnet--demo--train-models/           ← This repo.
│           └── .git/
│
└── externals/                                    ← Repositories you don't have write access to, but want to edit locally via symlinks or direct clones.
    ├── parnet_analyses_libs/
    ├── parnet_additional_utils/
    ├── parnet/
    └── metamotif/
```

#### Workspace organization with VSCode

Content of the `project-parnet-meta.code-workspace` file, which you can open in VSCode to load the whole workspace with all the repos:

```bash
{
  "folders": [
    {
      "path": "../meta"
    },
    {
      "path": "../parnet--demo/"
    },
  ],
  "settings": {
    // =========================
    // User settings
    // =========================
    "python.analysis.exclude": [
        "**/node_modules",
        "**/__pycache__",
        "**/.*",
        "**/resources/**",
        "**/data/**",
        "**/.snakemake/**",
        "**/results/**",
        "**/envs/**",
        "**/.pixi/**",
    ],
    "python.defaultInterpreterPath": "",
  },
  //
  "extensions": {
    // Recommended extensions for this workspace
    "recommendations": [
      // =========================
      // Python
      // =========================
      "ms-python.python", // Python support
      "ms-python.vscode-pylance", // Type checking (Pyright engine)
      "charliermarsh.ruff", // Linting + formatting (replaces Black, Flake8, isort)
      "njpwerner.autodocstring", // Auto-generate docstrings
      "ms-toolsai.jupyter", // Jupyter notebook support
      "ms-toolsai.jupyter-keymap", // Jupyter shortcuts
      "ms-toolsai.jupyter-renderers", // Rich output rendering
      "ms-toolsai.vscode-jupyter-cell-tags", // Cell tagging support
      // =========================
      // R
      // =========================
      "reditorsupport.r", // R language support
      // =========================
      // Snakemake / Workflow
      // =========================
      "tfehlmann.snakefmt", // Snakemake formatting
      // =========================
      // Documentation / Markdown
      // =========================
      "davidanson.vscode-markdownlint", // Markdown linting
      "ban.spellright", // Spell checking
      // =========================
      // Environment / Workspace
      // =========================
      "mkhl.direnv", // direnv integration
      "swellaby.workspace-config-plus", // Shared + local VSCode settings
      // =========================
      // Visual Code Helpers
      // =========================
      "usernamehw.errorlens", // Inline error highlighting
      "oderwat.indent-rainbow", // Indentation colorization
      "wayou.vscode-todo-highlight" // Highlight TODO / FIXME / NOTE
    ],
    "unwantedRecommendations": []
  }
}
```


### Organization of this repository

```bash
# $ tree -L 3 ../parnet--demo--train-models/
./parnet--demo--train-models/
├── config
│   ├── config.train_validation_test_split.yaml
│   ├── filepaths.lambosaur-ms-01-2.yaml
│   └── parnet_models_metadata.yaml
├── docs
│   ├── demo-full-set-filtered.md
│   ├── demo-spliceosome-hepg2.md
│   └── preprocess-source-parnet-datasets.md
├── externals
│   ├── metamotif -> /path/to/externals/lambosaur/metamotif/
│   ├── parnet_additional_utils -> /path/to/externals/parnet_additional_utils/
│   ├── parnet_analyses_libs -> /path/to/externals/parnet_analyses_libs/
│   └── pylbsr -> /path/to/externals/lambosaur/pylbsr/
├── LICENSE
├── notebooks
│   ├── demo--train-spliceosome-hepg2
│   │   ├── evaluate_retrained_models.py.ipynb
│   │   ├── prepare_datasets.py.ipynb
│   │   └── train_from_pretrained.py.ipynb
│   └── diagnostics
│       ├── explore_2000nt_datasets.py.ipynb
│       └── explore_600nt_hfds_source.py.ipynb
├── pixi.lock
├── pixi.toml
├── pyproject.toml
├── README.md
├── resources -> /path/to/parnet--demo--train-models/resources/
├── results -> /path/to/parnet--demo--train-models/results/
├── scripts
│   ├── convert_hfds_to_pt.py
│   ├── convert_pt_to_hfds.py
│   ├── export_bed.py
│   └── strip_padding.py
└── src
    └── parnet_demo_utils
        ├── __init__.py
        ├── bed_utils.py
        ├── datasets.py
        ├── filters.py
        ├── hfds_utils.py
        ├── sparse_utils.py
        └── training_utils.py
```


### Pixi environments

This requires that you install `pixi` as a user: <https://pixi.prefix.dev/latest/#installation>

All environments are defined in `pixi.toml`. Select one based on your GPU:

| Environment | CUDA | `parnet` source | When to use |
| --- | --- | --- | --- |
| `parnet-dev` | CPU only | git (pinned) | CPU-only machines, testing |
| `parnet-dev-local` | CPU only | editable (`externals/`) | Developing parnet on CPU |
| `parnet-dev-cu11` | CUDA 11 | git (pinned) | Standard GPU server (A100, V100…) |
| `parnet-dev-cu11-local` | CUDA 11 | editable (`externals/`) | Developing parnet on CUDA 11 GPU |
| `parnet-dev-cu12` | CUDA 12 | git (pinned) | RTX 5090 / sm_120 |
| `parnet-dev-cu12-local` | CUDA 12 | editable (`externals/`) | Developing parnet on CUDA 12 GPU |

Install the environment you need (takes a few minutes the first time):

```bash
pixi install -e parnet-dev-cu11        # or whichever variant fits your machine
```

Launch a JupyterLab server from within the environment:

```bash
pixi run -e parnet-dev-cu11 jupyter lab --no-browser --port 8888
```

### Local development setup (externals/)

The `-local` variants install sibling repos as editable packages via the `externals/`
directory, so code changes take effect immediately without reinstalling.
The symlinks are **not** committed to git — create them once after cloning:

```bash
cd externals/
ln -s /path/to/cloned/parnet_analyses_libs parnet_analyses_libs
ln -s /path/to/cloned/parnet_additional_utils parnet_additional_utils
ln -s /path/to/cloned/metamotif metamotif
```

Or clone any of these repos directly into `externals/<name>` instead of symlinking.

### Resources

Several large files are needed at runtime and are **not** committed to git.
On personal machines: you can place them directly under `resources/` or anywhere else and update the paths in `config/filepaths.yaml` accordingly.

On a shared server: since these files are large and shared across multiple users, we suggest to place them in a shared storage location (e.g. NAS) and symlink them into `resources/` as needed, or update the paths in `config/filepaths.yaml` to point to the shared location directly.

```bash
tar -xzf /path/to/local/models.tar.gz -C /path/to/shared_storage/resources/
tar -xzf /path/to/local/metadata.tar.gz -C /path/to/shared_storage/resources/

ln -s /mnt/nas/path/to/encode.filtered.hfds resources/parnet-encore-eclip/600nt_windows/encode.filtered.hfds
```

We also provide with pre-computed results for the spliceosome demo, which can be copied/unpacked similarly:

```bash
tar -xzf results/spliceosome-hepg2.precomputed.tar.gz -C results/
```

---

## Demos

### Preprocessing source datasets

The PARNET models were trained on "source datasets" generated at different timepoints of the project's history.

We focus on a *v1 600nt* dataset, containing 600nt tiles generated over the human transcriptome (GENCODE V40 annotations)
filtered for eCLIP signal (**tasks**) from any of the 223 ENCODE eCLIP experiments, from either eCLIP or SMI control **tracks**.

Check [docs/preprocess-source-parnet-datasets.md](docs/preprocess-source-parnet-datasets.md) for details on the source datasets, and how they were further preprocessed for this project.

The **canonical files** you should work with for any downstream work are the

```bash
# $ tree -L 2 ./resources/parnet-encore-eclip/600nt_windows.no-one-hot.stripped/
./resources/parnet-encore-eclip/600nt_windows.no-one-hot.stripped/
├── encode.filtered.bed
├── encode.filtered.hfds
│   ├── dataset_dict.json
│   ├── encode.filtered.metadata.yaml
│   ├── test
│   ├── train
│   └── valid
├── encode.filtered.hfds.tar
├── encode.filtered.hfds.tar.gz
├── encode.filtered.metadata.yaml
├── encode.filtered.pt
├── encode.filtered.pt.gz
└── full_rbp_set.tsv
```

See the [quick_inspect.py.ipynb](notebooks/diagnostics/quick_inspect.py.ipynb) notebook for a quick inspection of the canonical HFDS files.

### Spliceosome HepG2 (9-RBP subset)

End-to-end fine-tuning on a filtered subset of the ENCODE eCLIP data: dataset preparation →
optional format conversion → training → evaluation.

See [docs/demo-spliceosome-hepg2.md](docs/demo-spliceosome-hepg2.md) for the step-by-step walkthrough.

Notebooks: [`notebooks/demo--train-spliceosome-hepg2/`](notebooks/demo--train-spliceosome-hepg2/)

#### Expected resources content for spliceosome demo

Input resources:

```bash
./resources/
├── metadata/
│   ├── full_rbp_set.tsv
│   └── yeo_RBP_annotation.function.csv
├── models/
│   ├── parnet.21m-0.0.pt
│   └── parnet.7m-0.0.pt
└── parnet-encore-eclip/
    └── 600nt_windows.no-one-hot.stripped/
        ├── encode.filtered.bed
        ├── encode.filtered.hfds/
        ├── encode.filtered.metadata.yaml
        ├── encode.filtered.pt
        ├── encode.filtered.pt.gz
        └── full_rbp_set.tsv
```

Precomputed results:

```bash
./results/
├── spliceosome-hepg2.precomputed
│   ├── datasets
│   │   ├── dataset.metadata.yaml
│   │   ├── dataset.pt
│   │   ├── rbp_cts.tsv
│   │   └── tiles.bed
│   └── training
│       └── parnet.7m-0.0.ft-head.spliceosome-hepg2
│           ├── checkpoints
│           ├── csv_logs
│           ├── model.full.pt
│           ├── model.statedict.pt
│           ├── run_config.yaml
│           └── tensorboard_logs
└── spliceosome-hepg2.precomputed.tar.gz
```

---
