# Instructions for AI Coding Agents

This file contains specific instructions for AI coding agents (like Claude) working on this repository.

## Repository Context

This is a consolidated Docker image repository for MaNIAC Lab ML workloads. All dependencies are managed via Pixi (conda-forge + PyPI) and images are built via GitHub Actions.

## Critical Rules

### 1. Pixi Dependency Management

**ALWAYS commit both `pixi.toml` AND `pixi.lock` together.**

When modifying dependencies:
```bash
cd <image-directory>/
vim pixi.toml  # Make changes
CONDA_OVERRIDE_CUDA=12.6 pixi lock  # Regenerate lock file
git add pixi.toml pixi.lock  # Commit both
```

**NEVER:**
- Commit `pixi.toml` without regenerating `pixi.lock`
- Delete or gitignore `pixi.lock` (it ensures reproducibility)
- Modify `pixi.lock` manually

### 2. Dockerfile Commands

**ALWAYS prefix commands with `/app/entrypoint.sh` in the final stage.**

The entrypoint activates the Pixi environment. Without it, tools like `curl`, `git`, `python` won't be in PATH.

```dockerfile
# ✅ CORRECT
RUN /app/entrypoint.sh curl -O https://example.com/file
RUN /app/entrypoint.sh git clone https://github.com/user/repo
RUN /app/entrypoint.sh python -m pip install package

# ❌ WRONG - will fail with "command not found"
RUN curl -O https://example.com/file
RUN git clone https://github.com/user/repo
RUN python -m pip install package
```

### 3. Adding New Images - Complete Checklist

When adding a new image, ALL of these steps are required:

1. **Create directory structure:**
   ```bash
   mkdir -p <image-name>/config
   ```

2. **Create `pixi.toml`:**
   - Must include `[workspace]` section (not `[project]`)
   - Must include both `linux-64` and `osx-arm64` in `platforms`
   - Use `channels = ["conda-forge"]`

3. **Generate `pixi.lock`:**
   ```bash
   cd <image-name>/
   CONDA_OVERRIDE_CUDA=12.6 pixi install
   ```

4. **Create `Dockerfile`:**
   - Use multi-stage build pattern (see ml_platform/Dockerfile as template)
   - Use `ghcr.io/prefix-dev/pixi:noble-cuda-13.0.0` as base
   - Generate entrypoint via `pixi shell-hook`
   - Prefix all RUN commands in final stage with `/app/entrypoint.sh`

5. **Create `.dockerignore`:**
   ```
   .pixi/
   .git
   *.md
   ```

6. **Update workflow:**

   Add a new entry to the `matrix.include` array in `.github/workflows/build-images.yaml`:

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

   **Important:** Use the YAML block scalar `|-` for the `registries` field to ensure proper formatting. The `docker/metadata-action` expects a newline-separated string.

7. **Test locally:**
   ```bash
   docker build --platform linux/amd64 -t <image>:test <image>/
   docker run --rm <image>:test python --version
   ```

8. **Commit all files:**
   ```bash
   git add <image-name>/ .github/workflows/build-images.yaml
   git commit -m "feat: add <image-name> Docker image"
   ```

### 4. Modifying Existing Images

When changing dependencies or Dockerfile:

1. **Test locally BEFORE committing:**
   ```bash
   docker build --platform linux/amd64 -t <image>:test <image>/
   docker run --rm <image>:test <command-to-verify>
   ```

2. **Commit related files together:**
   - If changing dependencies: commit `pixi.toml` + `pixi.lock`
   - If changing Dockerfile: test that build succeeds
   - If changing config files: include them in same commit

3. **Use semantic commit messages:**
   - `feat:` for new capabilities
   - `fix:` for bug fixes
   - `chore:` for dependency updates
   - `refactor:` for restructuring without behavior change

### 5. Workflow Matrix Pattern

The `.github/workflows/build-images.yaml` uses a **static matrix** with an `include` array. All images are built on every trigger.

**Key points:**
- Matrix is statically defined in YAML using `matrix.include`
- ALL images are built on every trigger (push, PR, tag, manual)
- `registries:` uses YAML block scalar (`|-`) for clean multiline format - required by `docker/metadata-action`
- Matrix fields are accessed as `matrix.name`, `matrix.context`, etc. (not `matrix.image.name`)

**Build triggers:**
- **Push to main:** Build ALL images, push to registries
- **Git tag `v*`:** Build ALL images, push with version tags
- **Pull request:** Build ALL images, but don't push
- **Manual dispatch:** Build ALL images, push to registries

**When modifying workflow:**
- ALWAYS validate YAML syntax (use yamllint or IDE validation)
- NEVER hardcode secrets in workflow (use `${{ secrets.NAME }}`)
- Use `|-` for multiline `registries` field to ensure proper formatting

### 6. Common Mistakes to Avoid

❌ **Using `[project]` instead of `[workspace]` in pixi.toml**
- Modern pixi uses `[workspace]`, not `[project]`

❌ **Not prefixing commands with entrypoint**
- Leads to "command not found" errors in Docker build

❌ **Committing pixi.toml without pixi.lock**
- Breaks reproducibility

❌ **Forgetting to add new image to workflow matrix**
- Must add entry to `matrix.include` array in build-images.yaml

❌ **Not using block scalar for registries field**
- Use YAML block scalar: `registries: |-` with newlines, not inline strings or arrays

❌ **Not testing Docker build locally**
- CI failures waste time; test locally first

❌ **Adding dependencies to wrong section**
- Use `[dependencies]` for conda-forge packages
- Use `[pypi-dependencies]` for PyPI-only packages

### 7. Testing Requirements

Before committing changes that affect Docker builds:

1. **Build succeeds:**
   ```bash
   docker build --platform linux/amd64 -t <image>:test <image>/
   ```

2. **Environment activates:**
   ```bash
   docker run --rm <image>:test python --version
   ```

3. **Key packages import:**
   ```bash
   docker run --rm <image>:test python -c "import <package>; print('OK')"
   ```

4. **For ml_platform specifically:**
   ```bash
   docker run --rm ml-platform:test python -c "import tensorflow, keras, numpy, pandas; print('OK')"
   docker run --rm ml-platform:test root --version
   docker run --rm ml-platform:test jupyter --version
   ```

### 8. Repository State Awareness

**Key files to check before making changes:**
- `.github/workflows/build-images.yaml` - workflow matrix defines all images and configurations
- `<image>/pixi.toml` - dependency definitions
- `<image>/pixi.lock` - locked versions (DO NOT MODIFY MANUALLY)
- `<image>/Dockerfile` - build instructions

### 9. Build Behavior

The workflow builds **ALL images on every trigger** for simplicity and consistency.

**Build triggers:**
- **Push to main:** Build and push all images
- **Git tag `v*`:** Build and push all images with version tags
- **Pull request:** Build all images (validation only, no push)
- **Manual dispatch:** Build and push all images

**No change detection:** The workflow intentionally does not use path filters or conditional building. This simplifies maintenance and ensures all images stay in sync.

**When adding new images:** Simply add an entry to the `matrix.include` array in the workflow file. No additional configuration needed.

### 10. Git Workflow

**Commit frequently** with logical groupings:
- Dependency changes: pixi.toml + pixi.lock together
- New image: all files for that image + workflow update
- Workflow changes: standalone if not tied to image changes

**Semantic commit format:**
```
<type>: <short description>

<detailed explanation>

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Types:** feat, fix, chore, refactor, docs, test, ci

## When to Ask for Clarification

**ALWAYS ask the user before:**
- Removing or renaming an existing image
- Changing base image version (affects all downstream)
- Modifying registry configurations
- Adding new registry authentication requirements
- Changing CUDA version
- Making breaking changes to existing images

**You can proceed without asking when:**
- Adding new dependencies to pixi.toml (assuming you regenerate lock)
- Fixing obvious bugs in Dockerfile
- Improving documentation
- Adding new images (as long as you follow checklist)
- Updating non-breaking dependency versions

## Useful Commands Reference

```bash
# Generate/update pixi lock file
cd <image>/ && CONDA_OVERRIDE_CUDA=12.6 pixi install

# Test local build
docker build --platform linux/amd64 -t <image>:test <image>/

# Run verification tests
docker run --rm <image>:test <command>

# Interactive shell for debugging
docker run --rm -it <image>:test bash

# Check pixi environment
docker run --rm <image>:test pixi list

# View current git status
git status

# Check workflow syntax
yamllint .github/workflows/build-images.yaml
```

## Success Criteria

A change is complete when:
1. ✅ Docker build succeeds locally
2. ✅ Verification tests pass
3. ✅ Both pixi.toml and pixi.lock committed (if dependencies changed)
4. ✅ Workflow matrix updated (if new image added)
5. ✅ Commit message is semantic and descriptive
6. ✅ README.md updated if user-facing behavior changed

## Additional Context

This repository consolidates what were previously two separate repositories (ml_base and ml_platform) into a single monorepo with modern tooling:
- **Before:** apt-get + pip venv, separate repos, manual builds
- **After:** Pixi + conda-forge, monorepo, automated CI/CD

The goal is maintainability, reproducibility, and ease of adding new images.
