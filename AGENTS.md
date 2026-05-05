# AGENTS.md

## Project

`mri-normalization-tools` (`mnts` package) — MRI image normalization via a graph-based filter pipeline (bias field correction, spatial normalization, intensity normalization, binning).

## Commands

```bash
# Install (dev mode)
uv sync
uv pip install -e .

# Run all tests (must be in unit_test/ dir)
cd unit_test
python -m unittest discover -v

# Run a single test file
cd unit_test && python -m unittest test_filters.py

# Download test sample data first (required for most tests)
python unit_test/download_sample_data.py

# CLI tools (after install)
mnts-train --help
mnts-infer --help
mnts-dicom2nii --help
mnts-dcm-tagprint --help
```

## Architecture

```
mnts/
  filters/          Core filter pipeline
    geom/           SpatialNormalizer, ReorientFilter, GeomMaskCrop
    intensity/      N4BiasFieldCorrection, NyulNormalizer, ZScore, RangeRescale, SI rebinning, histogram matching
    mnts_filters.py       MNTSFilter, MNTSFilterRequireTraining, pipeline
    mnts_filters_graph.py  MNTSFilterGraph (graph-based filter DAG with YAML loading)
    mpi_wrapper.py         MPI parallel processing
    data_node.py           DataNode, TypeCastNode
  scripts/          CLI entry points (mnts-train, mnts-infer, mnts-dicom2nii, etc.)
  io/               DICOM formatting, Dixon image handling
  utils/            get_dicom_meta(), DICOM helpers
```

Filters subclass `MNTSFilter` and implement `filter(*args, **kwargs)`. Filters requiring training subclass `MNTSFilterRequireTraining`.

## Packaging: dual config, migration in progress

`pyproject.toml` and `setup.cfg` coexist. `pyproject.toml` is the newer one (uv-based) but `setup.cfg` has additional deps (`click`, `rich-tools`) and CLI entry points. Only edit `pyproject.toml` when adding dependencies unless the user says otherwise; keep `setup.cfg` in sync.

## Test setup gotchas

- Tests are `unittest.TestCase` (not pytest). Run from the `unit_test/` directory.
- Tests in `unit_test/` use `from utils import ...` — this is a local reference to `unit_test/utils.py`. Running from any other directory will fail.
- **Download sample data first**: `python unit_test/download_sample_data.py`. Tests that depend on NIfTI/DICOM data skip gracefully if the files are missing.
- Tests generate random data with SimpleITK shapes — reproducibility issues are expected with randomized inputs.

## Windows notes

- Scripts with multi-threading/MPI must include `if __name__ == '__main__':` guard to avoid recursive import loops on Windows.
- SimpleITK is imported as `import SimpleITK as sitk` (case-sensitive).

## YAML graph format

When creating filter graphs from YAML, each node uses the filter class name as the key. Extra args for `add_node()` go under the `_ext` key:

```yaml
NyulNormalizer:
    _ext:
        upstream: [0, 1]
        is_exit: True
```

## Python version

`.python-version` says 3.14 (uv.lock requires `>=3.14`), but `pyproject.toml` says `>=3.10` and `setup.cfg` supports 3.7–3.10. The lock file and `.python-version` are authoritative for local development.

## Logging

Custom logger via `MNTSLogger['name']`. Set global level with `MNTSLogger.set_global_log_level('debug')`. Call `logger.cleanup()` in `tearDown`.
