# README - PARNET Demo - Training Datasets and train models

## Overview

Repo aimed at demonstrating details about how to train PARNET.

Currently focusing on using pretrained models to speed up the process + training on filtered data; see the "spliceosome-hepg2" demo notebooks.

## Repo content

```bash
./resources/
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
    │   └── metadata.txt
    └── 600nt_windows
        └── encode.filtered.5.hfds.tar.gz
```

```bash
./results/
└── spliceosome-hepg2
    ├── dataset.pt
    ├── rbp_cts.tsv
    └── training
        ├── checkpoints
        │   ├── best.ckpt
        │   ├── best-v1.ckpt
        │   ├── last.ckpt
        │   └── last-v1.ckpt
        ├── csv_logs
        │   ├── version_0
        │   ├── version_1
        │   │   └── metrics.csv
        │   └── version_2
        │       └── metrics.csv
        ├── lightning_logs
        │   ├── version_0
        │   │   ├── events.out.tfevents.1779304153.ms-01-2.87877.0
        │   │   └── hparams.yaml
        │   └── version_1
        │       ├── events.out.tfevents.1779308542.ms-01-2.179953.0
        │       └── hparams.yaml
        ├── model.finetuned.pt
        ├── parnet.7m-0.0.ft-head.spliceosome-hepg2.full.pt
        └── parnet.7m-0.0.ft-head.spliceosome-hepg2.statedict.pt
```
