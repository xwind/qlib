# AGENTS.md

## Project

Qlib is an AI-oriented quantitative investment platform. Python package name: `pyqlib`. Python 3.8–3.12.

## Setup

Editable install requires two steps — Cython compilation then pip install:

```bash
make install          # prerequisite + dependencies (compiles .pyx → .so)
# or for full dev environment:
make dev              # prerequisite + all optional deps (dev, lint, docs, test, analysis, rl)
```

Cython modules live in `qlib/data/_libs/` (`rolling.pyx`, `expanding.pyx`). If `.so` files already exist, `make prerequisite` skips compilation.

Version is managed by `setuptools_scm` via git tags and written to `qlib/_version.py`. Never edit `_version.py` manually.

## Data Prerequisites

Most tests and `qrun` need market data downloaded first:

```bash
python scripts/get_data.py qlib_data --name qlib_data_simple --target_dir ~/.qlib/qlib_data/cn_data --interval 1d --region cn
```

RL tests need extra data: `python scripts/get_data.py download_data --file_name rl_data.zip --target_dir tests/.data/rl`

## Commands

### Lint (run in this order)

```bash
make black     # line length 120, excludes qlib/_version.py
make pylint    # heavily suppressed — see Makefile for disabled codes
make flake8    # ignores: E501,F541,E266,E402,W503,E731,E203
make mypy      # most of the codebase is excluded in .mypy.ini
make nbqa      # notebooks — requires black<26.1
```

Full lint: `make lint` (runs all five).

### Test

```bash
cd tests
python -m pytest . -m "not slow" --durations=0   # fast suite
python -m pytest . -m "slow" --durations=0        # slow suite (separate CI workflow)
python -m pytest qlib/tests/test_all_pipeline.py  # single pipeline test (from repo root)
```

- Tests live in root `tests/`, **not** `qlib/tests/` (which only holds helpers: `config.py`, `data.py`).
- RL tests are auto-skipped on non-Linux (`conftest.py`).

### Run a workflow

```bash
python qlib/cli/run.py examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml
# or: qrun <config.yaml> (after install)
```

### Build docs

```bash
make docs-gen
```

## Architecture

```
qlib/
  __init__.py    # qlib.init() — entry point for configuration
  config.py      # global config singleton
  data/          # data loading, ops (expression engine), storage backends, PIT data
    _libs/       # Cython: rolling.pyx, expanding.pyx
    dataset/     # Dataset and Handler classes
    ops.py       # expression operators ($close, Ref(), Mean(), etc.)
  model/         # base Model interface, ensemble, risk models, trainers
  contrib/       # contributed models (LightGBM, Transformer, etc.), ops, strategy, workflow
    model/       # <— add new benchmark models here
    workflow/
  workflow/      # experiment management, recorders, task templates
  backtest/      # backtesting engine, exchange, executor, positions
  strategy/      # trading strategy base classes
  rl/            # reinforcement learning framework (Linux only)
  cli/           # qrun entrypoint (run.py)
scripts/
  get_data.py    # download market data
  dump_bin.py    # convert data to qlib binary format
  data_collector/ # data collection scripts (Yahoo, BAO, etc.)
examples/
  benchmarks/    # model configs organized by model name
  benchmarks_dynamic/
tests/           # root-level test directory (run pytest from here)
```

## macOS Quirks

- pytest may segfault from OpenMP conflicts. Set `OMP_NUM_THREADS=1 MKL_NUM_THREADS=1 NUMEXPR_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 VECLIB_MAXIMUM_THREADS=1` before running.
- LightGBM requires `brew install libomp` and `pip install --no-binary=:all: lightgbm`.

## Commit Conventions

Conventional commits enforced via commitlint (`.commitlintrc.js`):
- Types: `build, chore, ci, docs, feat, fix, perf, refactor, revert, style, test, Release-As`
- Header max length: 100 characters

## Key Gotchas

- `setup.py` only handles Cython Extension definitions; `pyproject.toml` has all metadata and dependencies.
- `nbqa` requires `black<26.1` to avoid unfixable formatting issues in notebooks.
- `qrun` should be run from outside any directory named `qlib/` to avoid import confusion (e.g., `cd examples` first).
- The `qlib/contrib/` directory is excluded from mypy but is linted by pylint/flake8.
- docs build requires `scipy<=1.15.3` and `snowballstemmer<3.0`.
