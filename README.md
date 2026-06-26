# Evo2 Viral LoRA Training Data

This repository contains the minimal data handoff for Evo2 LoRA continue-pretraining on eukaryotic-host / broad-host viral genomes.

Use only the files in this repository for the first training run.

## Files

| Path | Use |
|---|---|
| `data/train.jsonl.gz` | Training JSONL. One viral genome or viral segment per line. |
| `data/valid.jsonl.gz` | Validation JSONL for checkpoint / hyperparameter selection. |
| `metadata/eukaryotic_host_core_manifest.tsv.gz` | Per-record metadata for audit and stratified evaluation. Not used directly by the training dataloader. |
| `savanna/preprocess_commands.sh` | Converts the JSONL files into Savanna/Evo2 indexed mmap datasets. |
| `savanna/data_config_viral_lora.json` | Dataset path snippet after preprocessing. |

## Dataset format

The JSONL files are gzip-compressed. Each line has this format:

```json
{"record": "DQ665917.1|ictv:VMR1000033", "text": "CCCCAAGCGCCCCCCCGGCGCCATCTCCG..."}
```

Only `text` is model input. `record` is a traceable sequence ID.

The nucleotide text has already been processed:

- RNA `U` was converted to `T`;
- sequences are uppercase;
- database/reference orientation is preserved;
- negative-sense RNA was not forcibly converted to coding sense;
- reverse-complement augmentation was not added;
- exact and near-duplicate reduction were applied;
- bacteria-host phage-like and archaea-host viruses were excluded.

For segmented viruses, each segment is one document. Segments from the same `genome_group_id` are kept in the same split.

## Current split

| Split | Records | Bases | Species | Human-priority records |
|---|---:|---:|---:|---:|
| train | 12,944 | 171,790,984 | 7,621 | 715 |
| valid | 1,349 | 17,266,753 | 848 | 36 |

There is no separate test set. The validation set is a species-holdout split relative to train.

After choosing the training recipe, the validation set can be folded back into train for a final adapter run. If this is done, do not report that same set as an unbiased validation/test set.

## Preprocess for Evo2/Savanna

From the root of this repository, run:

```bash
bash savanna/preprocess_commands.sh
```

The script runs:

```bash
python tools/preprocess_data.py \
  --input data/train.jsonl.gz \
  --output-prefix savanna/viral_euk_host_core_train \
  --tokenizer-type CharLevelTokenizer \
  --jsonl-keys text \
  --append-eod \
  --dataset-impl mmap
```

and the equivalent command for `data/valid.jsonl.gz`.

If `tools/preprocess_data.py` lives in a separate Evo2/Savanna repository, either:

1. copy this repository's `data/` and `savanna/` directories into that training repository, then run the script there; or
2. edit `savanna/preprocess_commands.sh` so `python tools/preprocess_data.py` points to the correct script path.

## Training config

After preprocessing, use these dataset prefixes in the Evo2/Savanna training config:

```json
{
  "train-data-paths": [
    "savanna/viral_euk_host_core_train_text_CharLevelTokenizer_document"
  ],
  "valid-data-paths": [
    "savanna/viral_euk_host_core_valid_text_CharLevelTokenizer_document"
  ],
  "test-data-paths": [],
  "tokenizer-type": "CharLevelTokenizer"
}
```

The same content is provided in:

```text
savanna/data_config_viral_lora.json
```

Use the preprocessed dataset prefix, not the original `.jsonl.gz`, as the training data path.

## Context length

The context length is not specified in `savanna/data_config_viral_lora.json`. That file only specifies dataset paths and tokenizer settings.

Set the context length in the Evo2/Savanna training config used by the training team. Depending on the exact config template, this field may be named `seq_length`, `seq-length`, `context_length`, or similar.

For the first LoRA model-selection run, use:

```text
context length = 16,384 tokens
```

If memory or throughput is limiting, `8,192` tokens is acceptable. If resources are comfortable, `32,768` tokens is a useful ablation. Do not use 1M context as the default first run; it is much more expensive and most records in this viral corpus are far shorter than 1M.

Please report the exact context length used together with the training loss curve and validation metrics.

## Recommended training use

- Start from an Evo2 long-context checkpoint as the base model.
- Use this corpus for LoRA continue-pretraining.
- Context length is a training configuration choice; do not pre-split the JSONL into fixed windows.

## Requested training report metrics

Please report only the following core metrics for the first training handoff:

1. Training loss curve.
   - Include steps or tokens seen on the x-axis.
   - Include learning rate if available.

2. Viral species-holdout validation loss / perplexity.
   - Report the overall value on `data/valid.jsonl.gz`.
   - Also report the same metric stratified by the manifest column `genome`, for example `ssRNA(+)`, `ssRNA(-)`, `dsRNA`, `dsDNA`, and `ssDNA`.

3. Base Evo2 versus viral LoRA comparison on the same validation set.
   - Evaluate the base Evo2 checkpoint on `data/valid.jsonl.gz`.
   - Evaluate the viral LoRA model on the same `data/valid.jsonl.gz`.
   - Report the absolute and relative change in validation loss / perplexity.
