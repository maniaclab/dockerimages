# MaNIAC Lab Docker Images

Consolidated Docker image repository using Pixi for dependency management and modern GitHub Actions CI/CD.

## Repository Structure

```
dockerimages/
├── .github/
│   └── workflows/
│       └── build-images.yaml    # CI/CD workflow with embedded image config
├── ml_platform/
│   ├── Dockerfile               # Multi-stage build with Pixi
│   ├── pixi.toml                # Dependency manifest (conda-forge + PyPI)
│   ├── pixi.lock                # Locked dependency versions
│   ├── .dockerignore            # Exclude .pixi cache from builds
│   └── config/
│       └── jupyter_notebook_config.py
└── README.md
```

## Images

- **[ml_platform](ml_platform/)** - Machine learning platform with Python 3.12, TensorFlow, Keras, ROOT, Jupyter, and HEP tools

See each image's subdirectory for detailed documentation, dependencies, and usage examples.

## Architecture

### Pixi-based Dependency Management

All dependencies (system packages, Python libraries, compilers, ROOT) are managed via [Pixi](https://pixi.sh/) and conda-forge, replacing traditional `apt-get` + `pip venv` workflows.

**Benefits:**
- Reproducible environments via `pixi.lock`
- Unified dependency resolution (no apt/pip conflicts)
- Binary packages from conda-forge (faster builds)
- Cross-platform lock files (Linux + macOS for development)

### Multi-Stage Docker Build

```dockerfile
# Stage 1: Build - install dependencies via pixi
FROM ghcr.io/prefix-dev/pixi:noble-cuda-13.0.0 AS build
RUN pixi install --locked
RUN pixi shell-hook > entrypoint.sh

# Stage 2: Final - copy environment only
FROM ghcr.io/prefix-dev/pixi:noble-cuda-13.0.0 AS final
COPY --from=build /app/.pixi/envs/default /app/.pixi/envs/default
COPY --from=build /app/entrypoint.sh /app/entrypoint.sh
```

**Key Points:**
- Pixi shell-hook generates entrypoint that activates environment
- All RUN commands in final stage use `/app/entrypoint.sh` prefix
- Singularity/Apptainer compatible via `/host-libs/` mount point

### CI/CD Workflow

Image configurations are defined directly in `.github/workflows/build-images.yaml` as a static matrix. All images are built on every trigger for simplicity and consistency:

```yaml
matrix:
  include:
    - name: ml_platform
      context: ./ml_platform
      dockerfile: ./ml_platform/Dockerfile
      registries: |-
        ghcr.io/maniaclab/ml-platform
        docker.io/ivukotic/ml_platform
        hub.opensciencegrid.org/usatlas/ml-platform
      platforms: linux/amd64
      build_args: CUDA_VERSION=12.6
```

**Triggers:**
- **Push to `main`:** Build ALL images → tag as `latest` + `sha-abc1234`
- **Git tag `v*`:** Build ALL images → tag as `X.Y.Z`, `X.Y`, `latest`, `sha-abc1234`
- **Pull request:** Build ALL images (no push, validation only)
- **Manual:** `workflow_dispatch` builds ALL images

**Multi-Registry Push:**
Authenticated via GitHub secrets:
- `GITHUB_TOKEN` (automatic) for ghcr.io
- `DOCKER_USERNAME` / `DOCKER_PASSWORD` for docker.io
- `OSG_HARBOR_ROBOT_USER` / `OSG_HARBOR_ROBOT_PASSWORD` for OSG Harbor

## Adding a New Image

### 1. Create Image Directory

```bash
mkdir -p new_image/config
cd new_image
```

### 2. Create `pixi.toml`

```toml
[workspace]
name = "new-image"
version = "1.0.0"
description = "Description here"
channels = ["conda-forge"]
platforms = ["linux-64", "osx-arm64"]

[dependencies]
python = "3.12.*"
numpy = "*"
# ... add dependencies

[pypi-dependencies]
# Packages not on conda-forge
some-package = "*"
```

### 3. Generate `pixi.lock`

```bash
CONDA_OVERRIDE_CUDA=12.6 pixi install
# This creates pixi.lock - commit both files
```

### 4. Create `Dockerfile`

```dockerfile
ARG CUDA_VERSION="12.6"
ARG ENVIRONMENT="default"

FROM ghcr.io/prefix-dev/pixi:noble-cuda-13.0.0 AS build
ARG CUDA_VERSION
ARG ENVIRONMENT
WORKDIR /app
COPY pixi.toml pixi.lock ./
ENV CONDA_OVERRIDE_CUDA=$CUDA_VERSION
RUN pixi install --locked --environment $ENVIRONMENT
RUN echo "#!/bin/bash" > /app/entrypoint.sh && \
    pixi shell-hook --environment $ENVIRONMENT -s bash >> /app/entrypoint.sh && \
    echo 'exec "$@"' >> /app/entrypoint.sh

FROM ghcr.io/prefix-dev/pixi:noble-cuda-13.0.0 AS final
ARG ENVIRONMENT
WORKDIR /app
COPY --from=build /app/.pixi/envs/$ENVIRONMENT /app/.pixi/envs/$ENVIRONMENT
COPY --from=build /app/pixi.toml /app/pixi.toml
COPY --from=build /app/pixi.lock /app/pixi.lock
COPY --from=build --chmod=0755 /app/entrypoint.sh /app/entrypoint.sh

# Add your custom setup here
RUN /app/entrypoint.sh python --version

ENTRYPOINT ["/app/entrypoint.sh"]
```

### 5. Create `.dockerignore`

```
.pixi/
.git
*.md
```

### 6. Update Workflow

Edit `.github/workflows/build-images.yaml` and add a new entry to the `matrix.include` array:

```yaml
- name: new_image
  context: ./new_image
  dockerfile: ./new_image/Dockerfile
  registries: |-
    ghcr.io/maniaclab/new-image
    docker.io/username/new-image
  platforms: linux/amd64
  build_args: CUDA_VERSION=12.6
```

**Important:** Use the YAML block scalar `|-` for the `registries` field to ensure proper formatting. The workflow builds ALL images on every trigger.

### 7. Test Locally

```bash
docker build --platform linux/amd64 -t new-image:test new_image/
docker run --rm new-image:test python --version
```

### 8. Commit and Push

```bash
git add new_image/ .github/workflows/build-images.yaml
git commit -m "feat: add new-image Docker image"
git push origin main
```

The CI will automatically build and push to all configured registries.

## Development Workflow

### Modifying Dependencies

1. Edit `<image>/pixi.toml`
2. Regenerate lock file: `cd <image> && CONDA_OVERRIDE_CUDA=12.6 pixi install`
3. Test locally: `docker build -t <image>:test <image>/`
4. Commit both `pixi.toml` and `pixi.lock`

### Testing Locally

```bash
# Build image
docker build --platform linux/amd64 -t <image>:test <image>/

# Verify environment activates
docker run --rm <image>:test python --version

# Interactive shell
docker run --rm -it <image>:test bash
```

### Releasing a Version

```bash
# Tag with semantic version
git tag -a v2026.2.0 -m "Release 2026.2.0"
git push origin v2026.2.0
```

This triggers a full build of all images with tags:
- `2026.2.0`
- `2026.2`
- `latest`
- `sha-abc1234`

## Maintenance

### Updating Base Image

The base image `ghcr.io/prefix-dev/pixi:noble-cuda-13.0.0` should be updated periodically:

1. Check for newer versions: https://github.com/prefix-dev/pixi-docker/pkgs/container/pixi
2. Update `FROM` lines in Dockerfiles
3. Test locally
4. Commit and push

### Updating Dependencies

Pixi automatically resolves the latest compatible versions unless pinned. To update:

```bash
cd <image>/
# Update pixi.toml with new version constraints
vim pixi.toml

# Regenerate lock file
CONDA_OVERRIDE_CUDA=12.6 pixi install

# Test
docker build -t <image>:test .

# Commit both files
git add pixi.toml pixi.lock
git commit -m "chore: update dependencies"
```

### Monitoring Builds

- **GitHub Actions:** https://github.com/maniaclab/dockerimages/actions
- **ghcr.io:** https://github.com/orgs/maniaclab/packages
- **OSG Harbor:** https://hub.opensciencegrid.org/harbor/projects

## Troubleshooting

### Pixi Installation Fails in Docker

**Error:** `Package not found` or dependency resolution fails

**Solution:** Check that:
1. Package exists on conda-forge: https://anaconda.org/conda-forge/<package>
2. Platform is `linux-64` (not `noarch` or `osx-arm64` only)
3. Move to `[pypi-dependencies]` if not on conda-forge

### Build Fails with "curl: not found"

**Error:** `/bin/sh: 1: curl: not found`

**Solution:** Prefix commands with entrypoint to activate environment:
```dockerfile
# Wrong
RUN curl -O https://example.com/file

# Correct
RUN /app/entrypoint.sh curl -O https://example.com/file
```

### Image Size Too Large

**Symptoms:** Image > 5GB

**Solutions:**
1. Use multi-stage build (already implemented)
2. Remove unused dependencies from `pixi.toml`
3. Add packages to `.pixi/.condapackageignore` to exclude caches
4. Use `--no-cache-dir` for pip in `[pypi-dependencies]`

## References

- **Pixi Documentation:** https://pixi.sh/
- **Matthew Feickert's SciPy 2024 Proceedings:** Pixi multi-stage Docker pattern
- **GitHub Actions:** https://docs.github.com/en/actions
- **Docker Build Push Action:** https://github.com/docker/build-push-action
- **Singularity GPU Support:** https://github.com/singularityware/singularity/issues/611

## License

[Add license information here]
