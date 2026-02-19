# Contributing to ml_platform

Thank you for your interest in contributing to ml_platform! This guide will help you set up your development environment and understand the contribution workflow.

## Development Environment Setup

### Prerequisites

- [Pixi](https://pixi.sh/) for dependency management
- Docker for testing image builds
- Git for version control

### Setting up the Development Environment

```bash
# Clone the repository
git clone https://github.com/maniaclab/ml_platform.git
cd ml_platform

# Install the development environment
pixi install -e dev
```

This installs the `dev` environment which includes:
- Python 3.11 (pinned to avoid warnings)
- tbump for version management
- All necessary development tools

## Making Changes

### Modifying Dependencies

Dependencies are managed via Pixi and split into features:

- **ml feature** (production): All ML packages, ROOT, Jupyter, etc.
- **dev feature** (development): tbump and other dev tools

To add a dependency:

1. Edit `pixi.toml`:
   ```toml
   [feature.ml.dependencies]
   new-package = "*"
   ```

2. Regenerate the lock file:
   ```bash
   CONDA_OVERRIDE_CUDA=12.6 pixi install
   ```

3. Test the changes:
   ```bash
   docker build -t ml-platform:test .
   docker run --rm ml-platform:test python -c "import new_package"
   ```

4. Commit both files:
   ```bash
   git add pixi.toml pixi.lock
   git commit -m "feat: add new-package dependency"
   ```

**Important:** Always commit `pixi.toml` and `pixi.lock` together.

### Testing Locally

Before committing changes that affect the Docker image:

```bash
# Build the image
docker build --platform linux/amd64 -t ml-platform:test .

# Test Python environment
docker run --rm ml-platform:test python --version

# Test key packages
docker run --rm ml-platform:test python -c "import tensorflow, keras, numpy, pandas; print('OK')"

# Test ROOT
docker run --rm ml-platform:test root --version

# Test Jupyter
docker run --rm ml-platform:test jupyter --version

# Interactive shell for manual testing
docker run --rm -it ml-platform:test bash
```

### Testing with GPU

If you have GPU access:

```bash
docker run --rm --gpus all ml-platform:test python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

## Version Management and Releases

This project uses **Calendar Versioning (CalVer)** with the format `YYYY.MM.DD`.

### Creating a Release

#### Quick Release (Recommended)

Use the current date to create a release:

```bash
pixi run -e dev bump
```

This automatically:
1. Updates version in `pixi.toml` and `tbump.toml`
2. Creates a git commit with message `Release YYYY.MM.DD`
3. Creates a git tag `vYYYY.MM.DD`
4. Pushes the tag to trigger CI/CD

#### Manual Release

For a specific date:

```bash
pixi run -e dev tbump 2026.02.19
```

#### Check Current Version

```bash
pixi run -e dev tbump current-version
```

### What Happens After a Release

When you push a tag (e.g., `v2026.02.19`), GitHub Actions will:
1. Build the Docker image
2. Push to all registries with tags:
   - `2026.02.19` (full CalVer)
   - `2026.02` (year-month)
   - `latest`
   - `sha-abc1234` (commit SHA)

## Pull Request Workflow

1. **Create a feature branch:**
   ```bash
   git checkout -b feat/your-feature-name
   ```

2. **Make your changes and test locally:**
   ```bash
   # Make changes
   vim pixi.toml

   # Test
   docker build -t ml-platform:test .
   docker run --rm ml-platform:test python -c "import your_package"
   ```

3. **Commit your changes:**
   ```bash
   git add pixi.toml pixi.lock
   git commit -m "feat: add your feature

   Detailed description of the change.

   Co-Authored-By: Your Name <your.email@example.com>"
   ```

4. **Push and create a pull request:**
   ```bash
   git push origin feat/your-feature-name
   ```

   Then create a PR on GitHub.

5. **CI will automatically:**
   - Build the Docker image
   - Validate the build (but won't push to registries)

## Commit Message Guidelines

Use semantic commit prefixes:

- `feat:` - New features or capabilities
- `fix:` - Bug fixes
- `chore:` - Dependency updates, maintenance
- `refactor:` - Code restructuring without behavior change
- `docs:` - Documentation updates
- `ci:` - CI/CD workflow changes
- `test:` - Test additions or updates

**Examples:**
```
feat: add scikit-image for image processing

Add scikit-image to ml feature dependencies for advanced
image processing capabilities in ML workflows.
```

```
fix: correct TensorFlow GPU configuration

Update TensorFlow GPU setup to properly detect CUDA devices
on systems with multiple GPUs.
```

## Common Development Tasks

### Update Base Image

```bash
# Edit Dockerfile
vim Dockerfile
# Change FROM lines to new version

# Test locally
docker build -t ml-platform:test .

# Commit
git add Dockerfile
git commit -m "chore: update base image to pixi:noble-cuda-13.1.0"
```

### Update Python Version

```bash
# Edit pixi.toml
vim pixi.toml
# Change python = "==3.12" to desired version

# Regenerate lock file
CONDA_OVERRIDE_CUDA=12.6 pixi install

# Test locally
docker build -t ml-platform:test .

# Commit
git add pixi.toml pixi.lock
git commit -m "chore: update Python to 3.13"
```

### Add a New Configuration File

```bash
# Add file to config/ directory
vim config/your_config.py

# Update Dockerfile to copy it
vim Dockerfile
# Add: COPY config/your_config.py /path/in/container/

# Test
docker build -t ml-platform:test .

# Commit
git add config/your_config.py Dockerfile
git commit -m "feat: add your_config configuration"
```

## Getting Help

- **Issues:** https://github.com/maniaclab/ml_platform/issues
- **MaNIAC Lab support:** See main README for contact information
- **CLAUDE.md:** Contains AI agent instructions and repository patterns

## Code Review Process

All pull requests require:
1. ✅ Successful Docker build
2. ✅ No merge conflicts
3. ✅ Both `pixi.toml` and `pixi.lock` committed together (if dependencies changed)
4. ✅ Clear, descriptive commit messages
5. ✅ Testing evidence (build logs, test outputs)

## Release Checklist

Before creating a release:

- [ ] All tests pass locally
- [ ] Docker image builds successfully
- [ ] Key packages import without errors
- [ ] `pixi.toml` and `pixi.lock` are in sync
- [ ] Commit messages follow guidelines
- [ ] Version number uses CalVer YYYY.MM.DD format

Then:

```bash
pixi run -e dev bump
git push origin main
git push origin v$(date +"%Y.%m.%d")
```

Monitor the build at: https://github.com/maniaclab/ml_platform/actions
