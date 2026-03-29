---
description: Fix unsafe model deserialization and harden AI/ML model usage
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(pip:*, python:*, modelscan:*, git:*, curl:*, which:*)
argument-hint: [directory]
---

Audit and harden AI/ML model usage for supply chain security. **This command takes action by default** - it fixes unsafe deserialization calls, flags dangerous model files, and adds hash verification. Changes are explained as they are made.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/checklist.md` for the full checklist (AI/ML section).
Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/tools.md` for tool options.
Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/ai-ml-configs.md` for implementation configurations.

## Detection

Find AI/ML model files and references in the project (or $1 if a directory is provided):

- `*.pkl` / `*.pickle` → Python pickle files
- `*.pt` / `*.pth` → PyTorch model checkpoints
- `*.h5` / `*.hdf5` → Keras/HDF5 models
- `*.onnx` → ONNX models (safe format)
- `*.safetensors` → SafeTensors (safe format)
- `*.joblib` → Joblib-serialized models
- References to `torch.load`, `pickle.load`, `joblib.load` in Python code
- References to `huggingface`, `transformers`, `diffusers` in dependencies

## Prerequisites

AI/ML model files or references were found. Check if `modelscan` is installed (`which modelscan`).

If **not installed** and unsafe model files (`.pkl`, `.pickle`, `.pt`, `.pth`, `.joblib`) were found, tell the user:

> **Recommended: install modelscan** — scans serialized ML models for hidden malicious code, supporting pickle, PyTorch, TensorFlow, and Keras formats.
>
> ```bash
> pip install modelscan
> ```
>
> Once installed, re-run this command to scan model files for embedded payloads. Continuing with code-level analysis for now.

If only safe formats (`.safetensors`, `.onnx`) or code references were found (no unsafe model files), skip this prerequisite — modelscan is not needed.

## Actions to Take

Work through these in order. **Make each change directly, explain what was done and why, then move to the next.**

### 1. Scan model files with ModelScan (CRITICAL — if installed and unsafe model files exist)

If modelscan is installed and `.pkl`, `.pickle`, `.pt`, `.pth`, or `.joblib` files were found, run it against each:

```bash
modelscan --path {file}
```

Parse and report findings. Any model file flagged as containing executable code should be treated as **CRITICAL** — it may contain an embedded payload that runs on `torch.load()` or `pickle.load()`.

### 2. Fix unsafe deserialization calls (CRITICAL)

Search all Python files and fix each occurrence:

- `torch.load(path)` → `torch.load(path, weights_only=True)`
  - Explain: without `weights_only=True`, torch.load executes arbitrary code embedded in the model file via pickle
- `pickle.load(f)` / `pickle.loads(data)` → flag prominently and suggest SafeTensors/ONNX alternatives
  - Do NOT just add a comment - if there's a safe alternative in the codebase, refactor to use it
  - If no safe alternative exists, add `# SECURITY: pickle deserialization executes arbitrary code - migrate to SafeTensors or ONNX`
- `joblib.load(path)` → flag and suggest alternatives (joblib uses pickle internally)
- `pd.read_pickle(path)` → flag and suggest `pd.read_parquet()` or `pd.read_csv()` alternatives

### 3. Flag committed model files (CRITICAL)

Flag any `.pkl`, `.pickle`, `.pt`, `.pth`, `.joblib` files committed to git:
- Add them to `.gitignore`
- Explain: binary model files should not be in git (use Git LFS, a model registry, or cloud storage)
- Note: don't delete the files - the user needs to migrate them first

### 4. Add hash verification to model downloads (HIGH)

Search for model download code (requests, urllib, wget, curl, `huggingface_hub.hf_hub_download`). For each:
- If downloading a model without verifying its hash, add hash verification after download
- For Hugging Face downloads, ensure `revision=` is set to a specific commit hash, not `main`
- Add a verification pattern:
  ```python
  import hashlib
  expected_hash = "sha256:..."
  actual_hash = hashlib.sha256(open(model_path, "rb").read()).hexdigest()
  assert actual_hash == expected_hash, f"Model hash mismatch: {actual_hash}"
  ```

### 5. Pin model sources (HIGH)

For Hugging Face model references:
- Ensure `revision=` parameter specifies a commit hash, not a branch name
- Resolve the current commit hash using the Hugging Face API: `curl -s https://huggingface.co/api/models/{org}/{model} | jq .sha`
- If the API query fails, add a `# TODO: pin revision to commit hash - check https://huggingface.co/{org}/{model}/commits/main` comment. **Never fabricate a model revision hash.**
- Add `trust_remote_code=False` where applicable (prevents executing arbitrary code from model repos)

## Output

After making all changes, provide a summary:

```
## AI/ML Hardening Summary

### Changes Made
- [x] Fixed N unsafe torch.load() calls (added weights_only=True)
- [x] Flagged N pickle.load() calls with migration guidance
- [x] Added .pkl/.pt files to .gitignore
- [x] Pinned N Hugging Face model references to specific revisions

### Manual Steps Needed
- [ ] Migrate [file] from pickle to SafeTensors format
- [ ] Move model files from git to [model registry / cloud storage]
- [ ] Add hash verification for model downloaded at [file:line]

### Recommended Next Steps
- If modelscan was not installed, install it (`pip install modelscan`) and re-run for deeper model scanning
- Set up a private model registry with access controls
- Consider sandboxed model loading environments
```
