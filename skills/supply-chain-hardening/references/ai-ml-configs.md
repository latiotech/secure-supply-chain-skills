# AI/ML Model Security Configurations

Practical configs for securing model loading, storage, and verification.

## Safe Deserialization Patterns

### PyTorch - Use weights_only=True

```python
# BAD - executes arbitrary code from the file
model = torch.load("model.pt")

# GOOD - only loads tensor data, blocks code execution
model = torch.load("model.pt", weights_only=True)

# GOOD - load state dict into a known architecture
model = MyModel()
model.load_state_dict(torch.load("model.pt", weights_only=True))
```

### Migrate from pickle to SafeTensors

```python
# Convert a PyTorch model to SafeTensors
from safetensors.torch import save_file, load_file

# Save
save_file(model.state_dict(), "model.safetensors")

# Load (no arbitrary code execution possible)
state_dict = load_file("model.safetensors")
model.load_state_dict(state_dict)
```

### Migrate from pickle to ONNX

```python
import torch

# Export PyTorch model to ONNX
dummy_input = torch.randn(1, 3, 224, 224)
torch.onnx.export(model, dummy_input, "model.onnx",
                   opset_version=17,
                   input_names=["input"],
                   output_names=["output"])

# Load ONNX model (safe - no code execution)
import onnxruntime as ort
session = ort.InferenceSession("model.onnx")
```

### Pandas - Avoid read_pickle

```python
# BAD - pickle deserialization
df = pd.read_pickle("data.pkl")

# GOOD - use safe formats
df = pd.read_parquet("data.parquet")
df = pd.read_csv("data.csv")
df = pd.read_feather("data.feather")
```

## Model Hash Verification

### Verify downloaded models

```python
import hashlib
from pathlib import Path

def verify_model(model_path: str, expected_hash: str) -> bool:
    """Verify model file integrity against a known-good hash."""
    sha256 = hashlib.sha256()
    with open(model_path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            sha256.update(chunk)
    actual = sha256.hexdigest()
    if actual != expected_hash:
        raise ValueError(
            f"Model hash mismatch!\n"
            f"  Expected: {expected_hash}\n"
            f"  Actual:   {actual}\n"
            f"  File may be tampered with."
        )
    return True
```

### Pin Hugging Face model revisions

```python
from transformers import AutoModel, AutoTokenizer

# BAD - pulls latest (mutable)
model = AutoModel.from_pretrained("bert-base-uncased")

# GOOD - pinned to specific commit
model = AutoModel.from_pretrained(
    "bert-base-uncased",
    revision="1a73582e3fa4c tried8e3c8e30b4afb0f5e8a4e3c8",  # pin to commit hash
    trust_remote_code=False,  # never execute remote code
)

# Resolve current commit hash
# curl -s https://huggingface.co/api/models/bert-base-uncased | jq .sha
```

### Pin Hugging Face Hub downloads

```python
from huggingface_hub import hf_hub_download

# BAD - downloads latest
path = hf_hub_download("org/model", "model.safetensors")

# GOOD - pinned revision
path = hf_hub_download(
    "org/model",
    "model.safetensors",
    revision="abc123def456",  # specific commit hash
)
```

## Model Scanning

### Picklescan

```bash
# Install
pip install picklescan

# Scan a single file
picklescan --path model.pkl

# Scan a directory
picklescan --path ./models/

# In CI pipeline
picklescan --path ./models/ --exit-code
```

### ModelScan

```bash
# Install
pip install modelscan

# Scan a model file
modelscan --path model.h5

# Scan multiple formats
modelscan --path ./models/
```

## .gitignore for Model Files

```gitignore
# Model files - use Git LFS, model registries, or cloud storage
*.pkl
*.pickle
*.pt
*.pth
*.h5
*.hdf5
*.joblib
*.ckpt
*.bin

# Keep safe formats if small enough for git
# !*.safetensors
# !*.onnx

# Model registries cache
.cache/huggingface/
.cache/torch/
```

## Git LFS for Model Files

If models must be versioned in git, use Git LFS:

```bash
# Install and initialize
git lfs install

# Track model file types
git lfs track "*.safetensors"
git lfs track "*.onnx"
git lfs track "*.pt"

# Verify .gitattributes was created
cat .gitattributes
```

```
# .gitattributes
*.safetensors filter=lfs diff=lfs merge=lfs -text
*.onnx filter=lfs diff=lfs merge=lfs -text
*.pt filter=lfs diff=lfs merge=lfs -text
```

## CI/CD Model Security Pipeline

```yaml
# GitHub Actions example
name: Model Security Scan
on:
  pull_request:
    paths:
      - 'models/**'
      - '*.py'

jobs:
  scan-models:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1

      - name: Install scanners
        run: pip install picklescan modelscan

      - name: Scan model files
        run: |
          find . -name "*.pkl" -o -name "*.pt" -o -name "*.pth" -o -name "*.h5" | while read f; do
            echo "Scanning: $f"
            picklescan --path "$f" --exit-code
            modelscan --path "$f"
          done

      - name: Check for unsafe load calls
        run: |
          # Fail if torch.load without weights_only=True
          if grep -rn 'torch\.load(' --include='*.py' | grep -v 'weights_only=True' | grep -v '#.*noqa'; then
            echo "::error::Found torch.load() without weights_only=True"
            exit 1
          fi
```

## Sandboxed Model Loading

For untrusted models that must use pickle (legacy systems), isolate the loading:

```python
import subprocess
import sys

def load_model_sandboxed(model_path: str):
    """Load a model in a subprocess with restricted permissions."""
    result = subprocess.run(
        [sys.executable, "-c", f"""
import torch
import json
model = torch.load("{model_path}", weights_only=True)
print(json.dumps({{"status": "ok", "keys": list(model.keys()) if isinstance(model, dict) else "non-dict"}}))
"""],
        capture_output=True,
        text=True,
        timeout=60,
    )
    if result.returncode != 0:
        raise RuntimeError(f"Model loading failed: {result.stderr}")
    return result.stdout
```

For stronger isolation, use container-based sandboxing:

```bash
# Run model loading in a container with no network
docker run --rm --network=none \
  -v ./model.pt:/model.pt:ro \
  python:3.12-slim \
  python -c "import torch; m = torch.load('/model.pt', weights_only=True); print('OK')"
```
