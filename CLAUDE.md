# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

maxATAC is a Python package that predicts transcription factor (TF) binding sites from ATAC-seq signal and DNA sequence in human cell types, using dilated convolutional neural networks (TensorFlow/Keras). Predictions are made at 32 bp resolution. It ships as a CLI (`maxatac`) with subcommands, published to PyPI. See the paper: Cazares et al. (2023) PLoS Comp Bio.

The three model inputs are: a `.2bit` DNA sequence file, a normalized ATAC-seq bigwig signal, and a trained `.h5` model. Reference genome is hg38 by default; other genomes require supplying custom chrom sizes, blacklist, and `.2bit` files.

## Commands

```bash
# Install for development (Python 3.9 required)
pip install -e .

# Download reference data + models to ~/opt/maxatac/data (required before most commands work)
maxatac data

# Run the CLI
maxatac -h
maxatac <subcommand> -h

# Tests: the runner downloads test fixtures (hg19.2bit + bigwigs) into tests/temp/ first, then runs pytest
cd tests && ./run_tests.sh

# Run pytest directly once fixtures exist in tests/temp/ (note --forked: each test in its own process,
# required because TensorFlow/CUDA state does not reset cleanly between tests)
pytest -v --cov=maxatac --forked tests/

# Run a single test file / test
pytest --forked tests/test_parser.py
pytest --forked tests/test_helpers.py::test_get_files
```

System dependencies (not pip-installable) must be on PATH: `bedtools`, `samtools`, `bedGraphToBigWig` (ucsc-bedgraphtobigwig), `pigz`, `wget`, `git`. Install via conda-forge/bioconda. See `INSTALL.md` and `README.md`.

## Architecture

**CLI dispatch.** `maxatac/bin/maxatac` is the entry point (installed as a `scripts` entry in `setup.py`, not a console_entry_point). It calls `parse_arguments()` in `maxatac/utilities/parser.py`, which builds one argparse subparser per subcommand and wires each to a `run_*` function via `set_defaults(func=...)`. `main()` then calls `args.func(args)`.

**Subcommand → analysis module** mapping. Each subcommand's logic lives in `maxatac/analyses/<name>.py` as a `run_<name>(args)` function:

| Subcommand  | Module                | Purpose |
|-------------|-----------------------|---------|
| `prepare`   | `prepare.py`          | BAM/fragments → normalized bigwig (wraps bash scripts in maxATAC_data) |
| `average`   | `average.py`          | Average multiple ATAC signal tracks |
| `normalize` | `normalize.py`        | Min-max-like normalization of a signal track |
| `train`     | `train.py`            | Train a DCNN model from a meta file |
| `predict`   | `predict.py`          | Predict TF binding → bigwig + thresholded BED |
| `benchmark` | `benchmark.py`        | Compare predictions to ChIP-seq gold standards |
| `peaks`     | `peaks.py`            | Call peaks on a maxATAC signal track |
| `threshold` | `threshold.py`        | Compute per-TF confidence thresholds |
| `variants`  | `variants.py`         | Sequence-specific (variant/allele) binding prediction |
| `data`      | `data.py`             | Download reference data + models |

**Analysis modules stay thin.** Heavy lifting lives in `maxatac/utilities/`, organized by concern: `training_tools.py`, `prediction_tools.py`, `prepare_tools.py`, `normalization_tools.py`, `benchmarking_tools.py`, `peak_tools.py`, `threshold_tools.py`, `variant_tools.py`, `genome_tools.py` (2bit/bigwig/chrom-size IO), `system_tools.py` (paths, versioning, `get_files`, `Mute`), `callbacks.py`, `plot.py`, `logger.py`.

**Model architecture.** `maxatac/architectures/dcnn.py` defines the dilated CNN (`get_dilated_cnn`), the custom `loss_function` (masked cross-entropy that ignores positions where `y_true < y_true_min`, i.e. blacklisted regions encoded as -1), and metrics (`pearson`, `dice_coef`). Input is 1024 bp windows; DNA is one-hot (4 channels) concatenated with ATAC signal → 5 input channels; output is 32 values (1024/32 bp resolution) through a sigmoid.

**Configuration constants** live in `maxatac/utilities/constants.py`: model hyperparameters (kernel sizes, dilation rates `[1,1,2,4,8,16]`, filter scaling, batch sizes), training defaults (20 epochs, 100 batches/epoch, Adam LR 1e-3), chromosome splits (`DEFAULT_TRAIN_CHRS` / `DEFAULT_VALIDATE_CHRS` / `DEFAULT_TEST_CHRS` — test is chr1+chr8, held out), and the data root path `~/opt/maxatac/data`. Change defaults here rather than hardcoding in analyses.

**Reference data is external.** Models (`.h5`), the hg38 genome, blacklist, chrom sizes, motif files, and the bash preprocessing scripts (`ATAC_bowtie2_pipeline.sh`, `scatac_generate_bigwig.sh`) are NOT in this repo — they live in the separate [maxATAC_data](https://github.com/MiraldiLab/maxATAC_data) repo and are fetched by `maxatac data` into `~/opt/maxatac/data`. Paths in `constants.py` assume that layout.

## Conventions that matter

- **The `Mute()` context manager** (`system_tools.py`) wraps TensorFlow/heavy imports throughout the codebase to suppress noisy import-time logging. Follow the existing pattern when adding imports of TF or maxATAC analysis modules.
- **Training meta file** is a TSV. Columns the training tools read include `Cell_Line`, `TF`, `ATAC_Signal_File`, `Binding_File`, `ATAC_Peaks`, `CHIP_Peaks`, and `Train_Test_Label` (rows are filtered to `== 'Train'`). See `training_tools.py`. Regions of interest (ROI) files are BED-like with `Chr | Start | Stop | ROI_Type | Cell_Line`.
- **Version** is resolved in `setup.py` from git tags → `maxatac/git_version` file, defaulting to the string in `get_version()`. Update the default there on release.
- **Windows note for tooling:** the runtime data pipeline shells out to Unix tools (bedtools, samtools, bedGraphToBigWig) and bash scripts, so the prepare/predict pipelines are intended to run on Linux/macOS (or the provided Docker image `miraldi/maxatac`), not natively on Windows.
