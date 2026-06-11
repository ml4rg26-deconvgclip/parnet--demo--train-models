# Preprocessing source PARNET datasets

This document explains the one-off preprocessing pipeline applied to the raw source
datasets before any task-specific filtering (e.g. spliceosome fine-tuning). Running it
once per machine produces:

- A local `.pt` copy suitable for fast mmap loading without a live NAS mount.
- Sidecar `.metadata.yaml` files recording the dataset structure (`task_names`,
  `total_key`, `n_tracks`, `seq_len`, split counts) so downstream scripts can discover
  the task layout without loading the full data.
- A normalised HFDS copy where sequences are stored as plain DNA strings (the original
  source HFDS may have sequences encoded as sparse one-hot tensors).


---

## Diagnostic notebooks

| Notebook | Dataset | Purpose |
|---|---|---|
| [`notebooks/diagnostics/explore_2000nt_datasets.py.ipynb`](../notebooks/diagnostics/explore_2000nt_datasets.py.ipynb) | 2000 nt | Data-quality exploration; documents known issues. |
| [`notebooks/diagnostics/explore_600nt_hfds_source.py.ipynb`](../notebooks/diagnostics/explore_600nt_hfds_source.py.ipynb) | 600 nt | Data-quality exploration; documents known issues. |

---

## Choosing a format — HFDS vs .pt on memory-constrained machines

| Format | RAM cost | Recommended when |
|---|---|---|
| `.hfds` (Arrow, streaming) | Minimal — no upfront load; HuggingFace streams shards on demand | Storage is fast or data is local; default for low-RAM machines |
| `.pt` uncompressed (mmap) | Minimal — OS pages in only the accessed regions (~300–500 MB) | Storage is slow or unavailable; working from a local copy |
| `.pt.gz` (compressed) | ~12–15 GB during decompression | Avoid on RAM-limited machines; keep only for archival/transfer |

**Rule of thumb:** never open `.pt.gz` directly on a machine with < 16 GB RAM.
Always unpack first:

```bash
gunzip -k resources/.../encode.filtered.pt.gz   # keeps the .gz; writes companion .pt
```

---

## Canonical dataset — 600 nt windows

Source : `resources/parnet-encore-eclip/600nt_windows_source/encode.filtered.hfds`

The 600 nt windows datasets exists in the form of a source HFDS (`.hfds` directory) where sequences are stored as one-hot tensors,
pre-padded to 600 nt with leading/trailing N's as needed.

Many metadata information are missing about the tiles, and so we reprocess here the source HFDS.
Different steps are applied:

1. Convertion to `.pt` format including a re-conversion of the sequences from one-hot tensors to DNA strings, and inference of padding side (`pad_side`) for each tile.
2. Reconversion of the `.pt` back to HFDS, now with sequences stored as DNA strings and a sidecar metadata file.
3. Postprocessing of the `.pt` to a stripped version where sequences are stored at native length without N-padding, and `pad_side` is stored in the metadata for load-time re-padding.
4. Conversion of the stripped `.pt` to HFDS for users who prefer HFDS format with native-length sequences and load-time padding.
5. For all formats, export of the BED6 file from the no-one-hot HFDS for easy reference and use in coordinate-based analyses.

### Step 1 — HFDS → .pt

Converts the source HFDS to a single PyTorch dict file and writes a sidecar
`encode.filtered.metadata.yaml` alongside it.

```bash
# Adapt paths.
PATH_SOURCE_FOLDER_HFDS="/mnt/storage-nas-fast-2/research/projects/hzm/parnet-analyses/MANUAL/parnet_encore_eclip/parnet_preprocessed_training_data/600nt_windows/encode.filtered.hfds/"
PATH_LOCAL=resources/parnet-encore-eclip/600nt_windows.no-one-hot.prepadded/

mkdir -p $PATH_LOCAL

time pixi run -e parnet-dev-cu11 python scripts/convert_hfds_to_pt.py \
    --input  $PATH_SOURCE_FOLDER_HFDS \
    --output $PATH_LOCAL/encode.filtered.pt \
    --metadata-basename encode.filtered \  # Will write encode.filtered.metadata.yaml alongside the .pt output.
    --total-key eCLIP \                    # Pin total_key explicitly; otherwise defaults to first task key.
    ;


# We compress with pigz for archival, but keep the uncompressed .pt for fast loading on low-RAM machines (mmap):
# NOTE: -k preserves the original .pt file, which is important for the low-RAM workflow.
pigz -p 16 -k $PATH_LOCAL/encode.filtered.pt
```

The script will produce both `encode.filtered.pt` and metadata `encode.filtered.metadata.yaml` alongside.

Time: ~40 minutes to convert.

NOTE: any notebook that loads `encode.filtered.pt.gz` will automatically prefer the
uncompressed `.pt` companion when it is present, using mmap — theoretically,
this cuts RAM from ~12–15 GB (full decompression) to ~300–500 MB.

### Step 2 — .pt.gz → canonical HFDS

```bash
PATH_LOCAL=resources/parnet-encore-eclip/600nt_windows.no-one-hot.prepadded/

# num-shards matches source (validation split is now named 'valid' in .pt)
time pixi run -e parnet-dev-cu11 python scripts/convert_pt_to_hfds.py \
    --input     $PATH_LOCAL/encode.filtered.pt \
    --outputdir $PATH_LOCAL/encode.filtered.hfds \
    --num-shards train:76,valid:18,test:10 \
    --metadata-basename encode.filtered.hfds \
    --total-key eCLIP \
    ;
```

Converts the `.pt` back to HFDS with sequences stored as plain DNA strings, and writes
a sidecar `encode.filtered.hfds.metadata.yaml` inside the HFDS directory.

`--num-shards train:76,valid:18,test:10` matches the original source sharding.

Time: ~30 minutes to convert.

### Step 3 — strip padding

Converts the `.pt` file from pre-padded format (sequences at 600 nt with leading /
trailing N's for short tiles) to native-length format (variable-length sequences with
`pad_side` in `meta` for load-time re-padding). This is the same format as the 2000 nt
dataset.

```bash
PATH_SOURCE=resources/parnet-encore-eclip/600nt_windows.no-one-hot.prepadded/
PATH_TARGET=resources/parnet-encore-eclip/600nt_windows.no-one-hot.stripped/

time pixi run -e parnet-dev-cu11 python scripts/strip_padding.py \
    --input  $PATH_SOURCE/encode.filtered.pt \
    --output $PATH_TARGET/encode.filtered.pt \
    --total-key eCLIP \
    ;
```

Writes `encode.filtered.metadata.yaml` alongside the output with
`is_pre_padded: false` and `seq_len: 600` (the model window; stored sequences are
shorter, at native tile length).

**Why strip?** Stripped datasets store only the genomic sequence content without
redundant N-padding. When loaded with `GzListDataset(..., length=600)`, the wrapper
class re-pads at batch time using the stored `pad_side`.

Time: ~20 minutes to strip padding.

NOTE: we also convert this file back to HFDS.


### Export BED6

This can be applied to all datasets.

```bash

for name in 600nt_windows.no-one-hot.prepadded 600nt_windows.no-one-hot.stripped; do
    PATH_LOCAL=resources/parnet-encore-eclip/$name/
    time pixi run -e parnet-dev-cu11 python scripts/export_bed.py \
        --input  $PATH_LOCAL/encode.filtered.pt \
        --format pt \
        --output $PATH_LOCAL/encode.filtered.bed \
        ;
done
```

Time: ~20 minutes on the `.pt` file.

---

## Metadata sidecar schema

Both scripts write a `.metadata.yaml` sidecar alongside their output. Example:

```yaml
sequence_format: string
signal_format:   sparse_tensor   # sparse_list for HFDS output
seq_len:         600
n_tracks:        223
task_names:      [eCLIP, control]
total_key:       eCLIP            # pass as total_key= to GzListDataset / HFDSDataset
splits:
  train: 348277 # Number of tiles ; here example numbers, likely not exact.
  valid: 57400
  test:  48301
source:          encode.filtered.hfds   # Note: path is machine-dependent
```

**Note on `--metadata-basename`:** if you run both scripts writing into the same
parent directory, use `--metadata-basename <name>` to give each sidecar a distinct name
and avoid overwriting:

```bash
python scripts/convert_hfds_to_pt.py  ... --metadata-basename encode.filtered.hfds-to-pt
python scripts/convert_pt_to_hfds.py  ... --metadata-basename encode.filtered.pt-to-hfds
```

## On renaming tracks

By default the output retains the source task names (`"eCLIP"`, `"control"`).
To align with parnet's internal convention (`"total"` for the main signal), pass:

```bash
--rename-track-names '{"eCLIP": "total"}'
```

Without this flag, callers must pass `total_key="eCLIP"` to `GzListDataset` /
`HFDSDataset` / `PreloadedListDataset`. With it, `total_key="total"` works universally.
See `parnet_additional_utils.patch_datasets` for the `total_key=` parameter.

### Interaction between `--rename-track-names` and `--total-key`

`--rename-track-names` is applied **before** `--total-key` is validated. This means
`--total-key` must name the key as it appears in the **output**, not the source:

```bash
# Source has "eCLIP"; rename to "total"; pin total_key as "total".
python scripts/convert_hfds_to_pt.py \
    --input  encode.filtered.hfds \
    --output encode.filtered.pt \
    --rename-track-names '{"eCLIP": "total"}' \
    --total-key total          # ← post-rename name; "eCLIP" here would be an error

# No rename; source key "eCLIP" is the output key; pin directly.
python scripts/convert_hfds_to_pt.py \
    --input  encode.filtered.hfds \
    --output encode.filtered.pt \
    --total-key eCLIP
```

If `--total-key` names a key that does not exist in the output, the script exits with
`ERROR: --total-key '...' not in task names [...]` before writing any data.


---

## Padding conventions — important notes

### Padding N's vs reference-genome N's

Not all leading/trailing N's in a sequence are padding inserted by parnet. Some tiles
from chromosome boundary regions have N's in the actual reference genome (assembly gaps,
pericentromeric sequence). The strip script identifies inserted padding purely from
coordinates: `total = 600 − (end − start)`. When `end − start == 600` the tile fills
the window exactly (`pad_side = −1`) and is returned unchanged — its edge N's are
reference genome sequence and must be preserved.

### On inferring `pad_side` in the source 600nt dataset

The source 600 nt HFDS is pre-padded (all sequences stored at 600 nt) but does not
carry `pad_side` in its per-tile metadata. `convert_hfds_to_pt.py` infers it for each
tile as follows:

1. Compute `total_pad = 600 − (end − start)` from the tile coordinates.
2. If `total_pad == 0`: the tile fills the window exactly. Any edge N's are reference-
   genome N's, not parnet-inserted. Assign `pad_side = -1` and store without inspecting
   the sequence.
3. If `total_pad > 0`: the tile is shorter than the window, so parnet added N's to reach
   600 nt. The edge N's are therefore safe to inspect. Call `classify_seq_padding(seq,
   strand)` which counts leading/trailing N's and maps them to the parnet convention
   (accounting for strand flip: on `−` strand, physical left corresponds to intent right):
   - Both edges have N's → `pad_side = 0` (center)
   - Only one edge has N's → `pad_side = 1` (left intent) or `2` (right intent)
4. Cross-validate: `infer_pad_sizes(name, pad_side, 600)` must reproduce the physical
   N counts; an `AssertionError` flags any mismatch.

**Key safety invariant:** physical N-count inspection is only used when `total_pad > 0`,
which guarantees the edge N's on the *padded* side originate from parnet. Full-window
tiles (`total_pad == 0`) are assigned `pad_side = -1` purely from coordinates, so
reference-genome N's at chromosome boundaries are never misclassified as padding.

**Residual edge case — tile with genomic N's on the *opposite* edge from parnet padding:**
If a tile is shorter than 600 nt (so parnet added N's on, say, the left) and its genomic
content also ends with reference-genome N's on the right, `classify_seq_padding` would
count N's on both sides and misclassify the tile as center-padded. The cross-validation
assertion (step 4) catches this: the inferred sizes from `infer_pad_sizes` would not
match the physical count and the script raises `AssertionError` rather than silently
writing a wrong `pad_side`. Conversion of the full 600 nt dataset completed without
triggering this assertion, confirming no such tiles exist in practice. This would require
a tile simultaneously at a chromosome boundary (short tile → `total_pad > 0`) and
abutting an uncertain-genome region on the opposite side — a genomically very unlikely
coincidence.


### On the 2000 nt dataset and old conventions

The "2000nt" source dataset is **not** prepadded, and `pad_side` was defined from tiling transcripts and
defining padding as the missing amount of nucleotides to reach 2000 nt at boundary tiles.

One important distinction with the post-/re-processed 600nt datasets is that in the 2000nt dataset,
tiles that fully fill the 2000 nt window-size were marked with `pad_side = 0`.
The padding process was never applied to these tiles (since they were already 2000 nt),
but they were marked as "center" for convenience since they don't have a padding side.

This is distinct from our current convention for the 600 nt dataset, where such "coordinates match required length"
tiles are assigned `pad_side=-1` (no padding).
Only tiles that actually require padding on both sides are marked as 0 (center).

This description is correct. The 2000 nt sequences are stored at full 2000 nt length
(pre-padded), and `pad_side` was set during dataset creation. Full-window tiles
(`end − start == 2000`) were given `pad_side = 0` as a placeholder, since no padding
direction is meaningful when no padding was added. Our 600 nt pipeline breaks this
ambiguity by reserving `pad_side = -1` exclusively for full-window tiles, keeping `0`
strictly for tiles that genuinely need bilateral padding.

### Center-pad convention

When `pad_side = 0` (center) and total padding is odd, the extra nucleotide is placed on
the **left** (ceil-left, floor-right). This matches the convention used to generate the
600 nt source dataset.

`parnet.data.datasets.ListDataset` uses the opposite convention (floor-left, ceil-right).
**Do not use `ListDataset` directly** for stripped data — use the wrapper classes:

```python
from parnet_additional_utils import GzListDataset, PreloadedListDataset

ds = GzListDataset("encode.filtered.stripped.pt", "train", length=600)
# GzListDataset auto-detects length=600 from the .metadata.yaml sidecar when present,
# so the explicit length= argument can be omitted if the sidecar is alongside the file.
```

`GzListDataset` and `HFDSDataset` auto-detect `length` from the sidecar `seq_len` when
not passed explicitly. `PreloadedListDataset` has no sidecar access — pass `length=600`
explicitly.

---

## Outdated dataset — 2000 nt windows

Source: `resources/parnet-encore-eclip/2000nt_windows/gencode.v48.annotation.transcripts.merged.tiles.data.filtered.splits.pt.gz`

- 223 RBP tracks; ~454 k tiles (train 348 k / valid 57 k / test 48 k)
- Sequences stored as 2000 nt DNA strings

**Why outdated:** We attempted using this dataset because of its `.pt` format.
Because the model operates on 600 nt windows, every downstream prepare
notebook using this source has to center-crop tiles to 600 nt (`CROP_START=700,
CROP_END=1300`).
This cropping is a hot-fix that makes each prepare notebook responsible for an extra step.

Since we reprocessed the 600nt source HFDS into `.pt` format + brought additional annotations,
we recommend using this reprocessed 600 nt dataset as the canonical source, rather than
trying to use the hot-fixed 2000nt-cropped-to-600nt dataset.

### Known issues in the 2000nt datasets Exploring datasets

Documented in detail in
[`notebooks/diagnostics/explore_2000nt_datasets.py.ipynb`](../notebooks/diagnostics/explore_2000nt_datasets.py.ipynb):

- **Coordinate offset:** tile coordinates follow 1-based GENCODE convention but are
  passed to a 0-based FASTA interface, causing a 1 nt off-by-one at tile boundaries.
- **Partial vs full tiles:** majority of tiles cover part of a transcript, very few cover a "full" transcript.
- **Shorter tiles and padding:** exploring the relation between border effects, full/partial tiles, and padding requirements.

---
