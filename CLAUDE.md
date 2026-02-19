# Instructions for AI Coding Agents

This file contains specific instructions for AI coding agents (like Claude) working on this repository.

## Repository Context

This is a Docker image repository for the MaNIAC Lab ML platform. All dependencies are managed via Pixi (conda-forge + PyPI) and the image is built via GitHub Actions.

## Critical Rules

### 1. Pixi Dependency Management

**ALWAYS commit both `pixi.toml` AND `pixi.lock` together.**

When modifying dependencies:
```bash
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

### 3. Modifying the Image

When changing dependencies or Dockerfile:

1. **Test locally BEFORE committing:**
   ```bash
   docker build --platform linux/amd64 -t ml-platform:test .
   docker run --rm ml-platform:test <command-to-verify>
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

### 4. Workflow Behavior

The `.github/workflows/build-images.yaml` builds the image on every trigger.

**Key points:**
- No matrix - all configuration is inlined
- Context is `.` (repo root), dockerfile is `./Dockerfile`
- Branch tagging enabled via `type=ref,event=branch`

**Build triggers:**
- **Push to main:** Build image, push with tags `main`, `latest`, `sha-abc1234`
- **Git tag `v*`:** Build image, push with tags `X.Y.Z`, `X.Y`, `sha-abc1234`
- **Pull request:** Build image (validation only, no push)
- **Manual dispatch:** Build image, push to registries

**Tag behavior:**

| Trigger | Tags |
|---------|------|
| Push to `main` | `main`, `latest`, `sha-abc1234` |
| Git tag `v2026.02.11` | `2026.02.11`, `2026.02`, `sha-abc1234` |
| Pull request | `sha-abc1234` (no push) |

**When modifying workflow:**
- ALWAYS validate YAML syntax (use yamllint or IDE validation)
- NEVER hardcode secrets in workflow (use `${{ secrets.NAME }}`)

### 5. Common Mistakes to Avoid

❌ **Using `[project]` instead of `[workspace]` in pixi.toml**
- Modern pixi uses `[workspace]`, not `[project]`

❌ **Not prefixing commands with entrypoint**
- Leads to "command not found" errors in Docker build

❌ **Committing pixi.toml without pixi.lock**
- Breaks reproducibility

❌ **Not testing Docker build locally**
- CI failures waste time; test locally first

❌ **Adding dependencies to wrong section**
- Use `[dependencies]` for conda-forge packages
- Use `[pypi-dependencies]` for PyPI-only packages

### 6. Testing Requirements

Before committing changes that affect Docker builds:

1. **Build succeeds:**
   ```bash
   docker build --platform linux/amd64 -t ml-platform:test .
   ```

2. **Environment activates:**
   ```bash
   docker run --rm ml-platform:test python --version
   ```

3. **Key packages import:**
   ```bash
   docker run --rm ml-platform:test python -c "import tensorflow, keras, numpy, pandas; print('OK')"
   docker run --rm ml-platform:test root --version
   docker run --rm ml-platform:test jupyter --version
   ```

### 7. Repository State Awareness

**Key files to check before making changes:**
- `.github/workflows/build-images.yaml` - workflow configuration
- `pixi.toml` - dependency definitions
- `pixi.lock` - locked versions (DO NOT MODIFY MANUALLY)
- `Dockerfile` - build instructions
- `config/jupyter_notebook_config.py` - Jupyter configuration

### 8. Build Behavior

The workflow builds the image on every trigger for simplicity and consistency.

**Build triggers:**
- **Push to main:** Build and push with `main`, `latest`, and SHA tags
- **Git tag `v*`:** Build and push with version tags (X.Y.Z, X.Y) and SHA tags
- **Pull request:** Build for validation (no push)
- **Manual dispatch:** Build and push

**No change detection:** The workflow intentionally does not use path filters. This simplifies maintenance and ensures builds stay consistent.

### 9. Git Workflow

**Commit frequently** with logical groupings:
- Dependency changes: pixi.toml + pixi.lock together
- Dockerfile changes: standalone if not tied to dependency updates
- Config file changes: include in relevant commit

**Semantic commit format:**
```
<type>: <short description>

<detailed explanation>

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Types:** feat, fix, chore, refactor, docs, test, ci

## When to Ask for Clarification

**ALWAYS ask the user before:**
- Changing base image version (affects entire build)
- Modifying registry configurations
- Adding new registry authentication requirements
- Changing CUDA version
- Making breaking changes to the image

**You can proceed without asking when:**
- Adding new dependencies to pixi.toml (assuming you regenerate lock)
- Fixing obvious bugs in Dockerfile
- Improving documentation
- Updating non-breaking dependency versions

## Useful Commands Reference

```bash
# Generate/update pixi lock file
CONDA_OVERRIDE_CUDA=12.6 pixi install

# Test local build
docker build --platform linux/amd64 -t ml-platform:test .

# Run verification tests
docker run --rm ml-platform:test <command>

# Interactive shell for debugging
docker run --rm -it ml-platform:test bash

# Check pixi environment
docker run --rm ml-platform:test pixi list

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
4. ✅ Commit message is semantic and descriptive
5. ✅ README.md updated if user-facing behavior changed

## Additional Context

This repository consolidates what were previously two separate repositories (ml_base and ml_platform) into a single repository with modern tooling:
- **Before:** apt-get + pip venv, separate repos, manual builds
- **After:** Pixi + conda-forge, single image repo, automated CI/CD

The goal is maintainability, reproducibility, and simplicity.
