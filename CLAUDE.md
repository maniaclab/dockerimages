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

6. **Update workflow in TWO places:**

   **A. Add change detection filter:**
   ```yaml
   changes:
     outputs:
       existing_image: ${{ steps.filter.outputs.existing_image }}
       new_image: ${{ steps.filter.outputs.new_image }}  # ← ADD THIS
     steps:
       - uses: dorny/paths-filter@v3
         with:
           filters: |
             existing_image:
               - 'existing_image/**'
             new_image:            # ← ADD THIS BLOCK
               - 'new_image/**'
   ```

   **B. Add matrix entry:**
   ```yaml
   matrix:
     image:
       - name: existing_image
         # ... config
       - name: new_image        # ← ADD THIS ENTIRE BLOCK
         context: ./new_image
         dockerfile: ./new_image/Dockerfile
         changed: ${{ needs.changes.outputs.new_image }}
         registries:
           - ghcr.io/maniaclab/new-image
           - docker.io/username/new-image
         platforms: linux/amd64
         build_args: |
           CUDA_VERSION=12.6
   ```

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

The `.github/workflows/build-images.yaml` uses a matrix to define images. Each matrix entry is a complete image definition.

**Key points:**
- `changed: ${{ needs.changes.outputs.<image> }}` links to change detection
- `registries:` is an array - gets joined with `\n` in metadata-action
- `build_args:` is multiline string with `|` syntax
- Build only runs if: `changed == 'true'` OR git tag OR manual trigger

**When modifying workflow:**
- ALWAYS validate YAML syntax (use yamllint or IDE validation)
- ALWAYS test change detection logic
- NEVER hardcode secrets in workflow (use `${{ secrets.NAME }}`)

### 6. Common Mistakes to Avoid

❌ **Using `[project]` instead of `[workspace]` in pixi.toml**
- Modern pixi uses `[workspace]`, not `[project]`

❌ **Not prefixing commands with entrypoint**
- Leads to "command not found" errors in Docker build

❌ **Committing pixi.toml without pixi.lock**
- Breaks reproducibility

❌ **Not updating both workflow locations when adding image**
- Must update both `changes` job outputs/filters AND matrix

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

### 9. Change Detection Logic

The workflow uses `dorny/paths-filter@v3` to detect changes:

```yaml
filters: |
  ml_platform:
    - 'ml_platform/**'
```

This means:
- Changes to `ml_platform/**` set `needs.changes.outputs.ml_platform = 'true'`
- The matrix entry checks `matrix.image.changed == 'true'`
- If true (or git tag or manual), the build runs

**When adding new images:**
- Add filter in `changes` job
- Add output in `changes.outputs`
- Reference in matrix entry as `changed: ${{ needs.changes.outputs.<image> }}`

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
4. ✅ Workflow updated correctly (if new image)
5. ✅ Change detection working (if new image)
6. ✅ Commit message is semantic and descriptive
7. ✅ README.md updated if user-facing behavior changed

## Additional Context

This repository consolidates what were previously two separate repositories (ml_base and ml_platform) into a single monorepo with modern tooling:
- **Before:** apt-get + pip venv, separate repos, manual builds
- **After:** Pixi + conda-forge, monorepo, automated CI/CD

The goal is maintainability, reproducibility, and ease of adding new images.
