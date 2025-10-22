---
description: Generate an opinionated, self-contained Dockerfile for a uv-based Python project
---

# Setup Python Dockerfile

Generate a ready-to-build Dockerfile using uv for dependency management. This command will produce a multi-stage Dockerfile example (minimal and production variants) and a detailed explanation. The generated Dockerfile is self-contained with inline examples and does not reference any external files other than the project's pyproject.toml and uv.lock which you should have in your repo.

When you run this command, it will ask for:
- Entrypoint (module path, e.g. your_package.entrypoint or -m module)
- Runtime system deps (comma separated, e.g. libpq-dev, build-essential)
- Template (minimal or production)

Minimal template example:

# Minimal multi-stage Dockerfile using uv
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS build
WORKDIR /app
ARG VERSION=0.0.0
ENV SETUPTOOLS_SCM_PRETEND_VERSION=${VERSION} UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
# Use cache mount for uv and bind pyproject + uv.lock for reproducible installs
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev
COPY . /app
# Re-run uv sync to ensure project and extras are installed into .venv
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-dev

FROM python:3.12-slim-bookworm AS runtime
WORKDIR /app
ENV PATH="/app/.venv/bin:$PATH" PYTHONUNBUFFERED=1
# Optional: install runtime system deps
RUN apt-get update && apt-get install -y --no-install-recommends tini ${RUNTIME_APT} && apt-get clean && rm -rf /var/lib/apt/lists/*
# Create non-root user
RUN useradd --create-home --shell /bin/bash app
USER app
COPY --from=build --chown=app:app /app/.venv /app/.venv
COPY --from=build --chown=app:app /app/src /app/src
WORKDIR /app/src
ENTRYPOINT ["tini", "--"]
CMD ["python", "-m", "your_package.entrypoint"]

Production template example:

# syntax=docker/dockerfile:1
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS build
WORKDIR /app
ARG VERSION=0.0.0
ENV SETUPTOOLS_SCM_PRETEND_VERSION=${VERSION} UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
# Mount uv cache and bind pyproject/lock to make deterministic build
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev
# Copy everything and sync again to pick up project sources and extras
COPY . /app
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-dev --extra server

FROM python:3.12-slim-bookworm AS runtime
ARG VERSION
ARG VCS_REF
ARG BUILD_DATE
LABEL org.opencontainers.image.version="$VERSION" \
      org.opencontainers.image.revision="$VCS_REF" \
      org.opencontainers.image.created="$BUILD_DATE"
RUN apt-get update && apt-get install -y --no-install-recommends tini ${RUNTIME_APT} && apt-get clean && rm -rf /var/lib/apt/lists/*
RUN useradd --create-home --shell /bin/bash app
USER app
WORKDIR /home/app
ENV PATH="/home/app/.venv/bin:$PATH" PYTHONUNBUFFERED=1
COPY --from=build --chown=app:app /app/.venv /home/app/.venv
COPY --from=build --chown=app:app /app/src /home/app/src
ENTRYPOINT ["tini", "--"]
CMD ["/home/app/.venv/bin/python", "-m", "your_package.entrypoint"]


Notes and guidance:
- Enable BuildKit (DOCKER_BUILDKIT=1) when building to allow cache mounts.
- Prefer copying the .venv from build stage to runtime so runtime image is deterministic and doesn't run pip installs.
- Use tini or dumb-init as ENTRYPOINT to avoid orphaned processes.
- If you need helper scripts to install native libs, generate them in the build stage and COPY them into runtime.

Placeholders like ${RUNTIME_APT} and the entrypoint will be replaced with the values you provide when invoking the command.
