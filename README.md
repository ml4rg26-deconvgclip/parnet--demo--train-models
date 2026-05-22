# README - PARNET Demo - Training Datasets and train models

## Overview

Repo aimed at demonstrating details about how to train PARNET.

Currently focusing on using pretrained models to speed up the process + training on filtered data; see the "spliceosome-hepg2" demo notebooks.

## Repo content

```bash
resources
├── metadata
│   ├── full_rbp_set.tsv
│   └── yeo_RBP_annotation.function.csv
├── metadata.tar.gz
├── models
│   ├── parnet.21m-0.0.pt
│   └── parnet.7m-0.0.pt
├── models.tar.gz
└── parnet-encore-eclip
    ├── 2000nt_windows
    │   ├── gencode.v48.annotation.transcripts.merged.tiles.data.filtered.splits.pt.gz
    │   ├── gencode.v48.annotation.transcripts.merged.tiles.data.filtered.splits.pt
    │   └── metadata.txt
    └── 600nt_windows
        ├── encode.filtered.5.hfds.tar.gz
        └── encode.filtered.hfds/
```

```bash
./results/
└── spliceosome-hepg2
    ├── datasets
    │   ├── dataset.pt
    │   └── rbp_cts.tsv
    ├── results__spliceosome-hepg2__datasets.tar.gz
    └── training
        └── parnet.7m-0.0.ft-head.spliceosome-hepg2
            ├── checkpoints
            │   ├── best.ckpt
            │   └── last.ckpt
            ├── csv_logs
            │   └── version_0
            │       └── metrics.csv
            ├── model.full.pt
            ├── model.statedict.pt
            ├── run_config.yaml
            └── tensorboard_logs
                └── version_0
                    ├── events.out.tfevents.1779436911.ms-01-2.436666.0
                    └── hparams.yaml
```
