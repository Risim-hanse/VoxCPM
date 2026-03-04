# Python 3.12 support & `uv` dependency management (VoxCPM)

## Scope
This note documents whether VoxCPM can support **Python 3.12**, what the *dependency chain* looks like, and what repo changes are required to make 3.12 a supported runtime when using **`uv`**.

## Current packaging state (repo snapshot)
- Package metadata lives in `pyproject.toml` (PEP 621).
- `requires-python = ">=3.10"` (already includes 3.12).
- The repo depends on a mix of:
  - **GPU/CPU native wheels** (PyTorch stack)
  - **C/C++/Rust extension wheels** (e.g. `tokenizers`, `sentencepiece`, `pyarrow`, `scipy`)
  - **Pure-Python packages** (e.g. `einops`, `tqdm`)

## Dependency tree (direct deps + important transitive deps)
The project’s direct dependencies (from `pyproject.toml`) are:

- `torch`, `torchaudio`, `torchcodec`
- `transformers`, `einops`, `safetensors`
- `gradio`, `spaces`
- `inflect`, `addict`, `wetext`
- `modelscope`, `datasets`, `huggingface-hub`, `pydantic`, `tqdm`
- `simplejson`, `sortedcontainers`, `soundfile`, `librosa`, `matplotlib`
- `funasr`, `argbind`

A representative dependency tree (captured via `uv tree --python-version 3.12 --depth 2`) shows:

- `torch` → `triton` + NVIDIA CUDA runtime libs (when GPU wheels are selected)
- `transformers` → `tokenizers` (Rust extension) + `safetensors`
- `datasets` → `pyarrow` (C++ extension) + `pandas` + `numpy`
- `librosa` → `numba` → `llvmlite` (native)
- `wetext` → `kaldifst` (native)
- `funasr` → `modelscope` + `sentencepiece` + `scipy` + `tensorboardx`

## Python 3.12 compatibility: findings
### 1) Resolver check (packaging metadata)
A full resolve using Python 3.12 succeeds:

```bash
uv python install 3.12
uv lock --python python3.12 --dry-run
```

This indicates the declared requirements could be resolved and locked for the tested environment (Python 3.12, current index state), at least at the metadata level.
### 2) Code scan for common 3.12 breakages
A quick scan of `src/voxcpm/**` did **not** find typical 3.12 blockers like:
- `distutils` imports (removed)
- `collections.Mapping`-style deprecated imports
- `imp` usage

### 3) Practical risks (native wheels / platform matrix)
Even if dependency resolution succeeds, Python 3.12 support can still be blocked by *platform wheel availability* (or requiring local compilation).

Packages to pay special attention to when claiming “3.12 supported”:

- **PyTorch stack**: `torch`, `torchaudio`, `torchcodec`
  - Usually fine on 3.12, but availability differs by OS/CUDA version.
  - `torchcodec` also requires FFmpeg shared libraries at runtime (e.g. installing `ffmpeg` on Ubuntu). Without them, `import torchcodec` will fail even if the wheel installs successfully.
- **Rust/C++ extensions**: `tokenizers`, `sentencepiece`, `orjson`, `xxhash`, `pyahocorasick`, `editdistance`
- **Scientific stack**: `numpy`, `scipy`, `pyarrow`
- **JIT stack**: `numba`, `llvmlite` (often the slowest to catch up on new Python versions)
- **Kaldi/OpenFST**: `kaldifst` (wheel availability can vary by platform)

If you need Windows support, `kaldifst` is the first package to verify.

## Repo changes for “official” Python 3.12 support
Already applied in this repo (files):
- `pyproject.toml` (Python 3.12 classifier; Black targets)
- `uv.lock` (locked dependency graph)
- `.gitignore` (ignore `.venv/`, `.uv/`)
- `docs/python312_uv.md` (this document)

Details:

1. **Declare Python 3.12 support in package metadata**
   - Added `"Programming Language :: Python :: 3.12"` classifier in `pyproject.toml`.

2. **Adopt `uv` locking**
   - Added `uv.lock` generated via:
     ```bash
     uv lock --python python3.12
     ```

3. **Ignore uv/venv directories**
   - Updated `.gitignore` to include `.venv/` and `.uv/`.

## How to work with this repo using `uv`
### Setup
```bash
# 1) Ensure a 3.12 interpreter exists
uv python install 3.12

# 2) Create a local virtualenv (default: .venv)
uv venv --python 3.12

# 3) Install locked dependencies
uv sync

# (optional) Dev extras
uv sync --extra dev
```

### Run
```bash
# Run the CLI
uv run voxcpm --help

# Run the web demo
uv run python app.py
```

### Update dependencies
```bash
# Update lockfile (respect existing pins)
uv lock

# Upgrade everything (or a single package)
uv lock --upgrade
uv lock --upgrade-package transformers
```

## Validation checklist (recommended)
Minimum:
- `uv run python -c "import voxcpm"`
- `uv run python -c "import torch, torchaudio"`

Suggested (since many issues are runtime/native):
- Run the README "Basic Usage" example on Python 3.12.
- Run `uv run python app.py` and perform one synthesis request.
- If you rely on fine-tuning scripts, run a short smoke test (imports + argument parsing) under 3.12.

## Notes / follow-ups
- There is no CI workflow that tests runtime compatibility across Python versions. If you want to *guarantee* 3.12 support, add a GitHub Actions job that runs a basic import + (optional) a small inference smoke test for Python 3.10–3.12.
