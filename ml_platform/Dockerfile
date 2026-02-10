ARG CUDA_VERSION="12.6"
ARG ENVIRONMENT="default"

# ============================================================
# Stage 1: Build - install all dependencies via pixi
# ============================================================
FROM ghcr.io/prefix-dev/pixi:noble-cuda-13.0.0 AS build

ARG CUDA_VERSION
ARG ENVIRONMENT

WORKDIR /app
COPY pixi.toml pixi.lock ./
ENV CONDA_OVERRIDE_CUDA=$CUDA_VERSION
RUN pixi install --locked --environment $ENVIRONMENT

# Generate entrypoint from pixi shell-hook
RUN echo "#!/bin/bash" > /app/entrypoint.sh && \
    pixi shell-hook --environment $ENVIRONMENT -s bash >> /app/entrypoint.sh && \
    echo 'exec "$@"' >> /app/entrypoint.sh

# ============================================================
# Stage 2: Final runtime image
# ============================================================
FROM ghcr.io/prefix-dev/pixi:noble-cuda-13.0.0 AS final

ARG ENVIRONMENT

WORKDIR /app

# Copy pixi environment (the only large layer)
COPY --from=build /app/.pixi/envs/$ENVIRONMENT /app/.pixi/envs/$ENVIRONMENT
COPY --from=build /app/pixi.toml /app/pixi.toml
COPY --from=build /app/pixi.lock /app/pixi.lock
COPY --from=build /app/.pixi/.gitignore /app/.pixi/.gitignore
COPY --from=build /app/.pixi/.condapackageignore /app/.pixi/.condapackageignore
COPY --from=build --chmod=0755 /app/entrypoint.sh /app/entrypoint.sh

# Singularity/Apptainer host GPU driver compatibility
# see https://github.com/singularityware/singularity/issues/611
RUN mkdir -p /host-libs && \
    echo "/host-libs/" > /etc/ld.so.conf.d/000-host-libs.conf && \
    ldconfig

# Jupyter configuration
COPY config/jupyter_notebook_config.py /usr/local/etc/jupyter_notebook_config.py

# Workspace directory
RUN mkdir -p /workspace

# Register Python kernel for Jupyter
RUN /app/entrypoint.sh python -m ipykernel install --name py312 --display-name "Python 3.12"

# User sync script (MaNIAC Lab infrastructure)
RUN /app/entrypoint.sh curl -fsSL -o /usr/local/bin/sync_users_debian.sh \
    https://raw.githubusercontent.com/maniaclab/ci-connect-api/master/resources/provisioner/sync_users_debian.sh && \
    chmod +x /usr/local/bin/sync_users_debian.sh

# ML platform tests
RUN /app/entrypoint.sh git clone https://github.com/maniaclab/ML_platform_tests.git /workspace/ML_platform_tests

# Build metadata
RUN echo "Timestamp: $(date --utc)" | tee /image-build-info.txt

ENTRYPOINT ["/app/entrypoint.sh"]
