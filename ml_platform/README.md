# ml_platform

Machine learning platform with Python 3.12, TensorFlow, Keras, ROOT, Jupyter, and HEP tools.

## Images

**Registries:**
- `ghcr.io/maniaclab/ml-platform`
- `docker.io/ivukotic/ml_platform`
- `hub.opensciencegrid.org/usatlas/ml-platform`

**Base:** `ghcr.io/prefix-dev/pixi:noble-cuda-13.0.0` (Ubuntu 24.04 + CUDA 13.0)

**Platforms:** linux/amd64

## Dependencies

All dependencies are managed via Pixi (conda-forge + PyPI). See [`pixi.toml`](pixi.toml) for the complete list and version constraints.

**Highlights:**
- Python 3.12, ROOT 6.32+, OpenJDK 8
- ML frameworks: TensorFlow, Keras, scikit-learn
- Data science: NumPy, Pandas, SciPy, PyArrow, HDF5
- Jupyter ecosystem: JupyterLab, ipywidgets, jupyterlab-git, RISE
- HEP tools: uproot, atlasify, rucio-jupyterlab
- Visualization: Matplotlib, Seaborn, Bokeh

## Usage

### Pull Image

```bash
# From GitHub Container Registry
docker pull ghcr.io/maniaclab/ml-platform:latest

# From Docker Hub
docker pull ivukotic/ml_platform:latest

# From OSG Harbor
docker pull hub.opensciencegrid.org/usatlas/ml-platform:latest
```

### Run Interactive Shell

```bash
docker run --rm -it ghcr.io/maniaclab/ml-platform:latest bash
```

### Run Jupyter Lab

```bash
docker run --rm -p 9999:9999 ghcr.io/maniaclab/ml-platform:latest jupyter lab --ip=0.0.0.0 --port=9999
```

Then open http://localhost:9999 in your browser.

### Run with GPU Support

```bash
docker run --rm --gpus all -it ghcr.io/maniaclab/ml-platform:latest python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

### Mount Data Volume

```bash
docker run --rm -v /path/to/data:/data -it ghcr.io/maniaclab/ml-platform:latest bash
```

### Singularity/Apptainer

```bash
singularity pull docker://ghcr.io/maniaclab/ml-platform:latest
singularity run ml-platform_latest.sif python --version
```

## Features

### Pixi Environment

All packages are managed via Pixi and activated automatically via the entrypoint. No need to source activation scripts.

### Jupyter Configuration

- Binds to `0.0.0.0:9999` by default
- Password change disabled (for container security)
- Browser auto-open disabled

### GPU Support

- CUDA 13.0 base image
- Singularity/Apptainer GPU driver compatibility via `/host-libs/` mount
- TensorFlow compiled with GPU support

### User Management

Includes `sync_users_debian.sh` for MaNIAC Lab user synchronization infrastructure.

### ML Platform Tests

Pre-installed at `/workspace/ML_platform_tests/` for validation and tutorials.

## Development

### Modifying Dependencies

1. Edit `pixi.toml`
2. Regenerate lock file:
   ```bash
   cd ml_platform/
   CONDA_OVERRIDE_CUDA=12.6 pixi lock
   ```
3. Test locally:
   ```bash
   docker build -t ml-platform:test .
   ```
4. Commit both files:
   ```bash
   git add pixi.toml pixi.lock
   git commit -m "chore: update dependencies"
   ```

### Testing Locally

```bash
# Build
docker build --platform linux/amd64 -t ml-platform:test .

# Test Python
docker run --rm ml-platform:test python --version

# Test ML packages
docker run --rm ml-platform:test python -c "import tensorflow, keras, numpy, pandas; print('OK')"

# Test ROOT
docker run --rm ml-platform:test root --version

# Test Jupyter
docker run --rm ml-platform:test jupyter --version

# Test HEP tools
docker run --rm ml-platform:test python -c "import uproot, atlasify; print('OK')"
```

## Troubleshooting

### Import Errors

If you get `ModuleNotFoundError`, ensure the package is in `pixi.toml`:
- Check if package exists on conda-forge: https://anaconda.org/conda-forge/<package>
- If not, add to `[pypi-dependencies]` section instead

### GPU Not Detected

```bash
# Check CUDA is visible
docker run --rm --gpus all ml-platform:test nvidia-smi

# Check TensorFlow GPU support
docker run --rm --gpus all ml-platform:test python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

### Singularity GPU Issues

Ensure `/host-libs/` is bound to host driver paths:
```bash
singularity run --nv --bind /usr/lib/x86_64-linux-gnu:/host-libs ml-platform_latest.sif
```

## References

- [Pixi Documentation](https://pixi.sh/)
- [TensorFlow GPU Support](https://www.tensorflow.org/install/gpu)
- [ROOT Documentation](https://root.cern/)
- [JupyterLab Documentation](https://jupyterlab.readthedocs.io/)
