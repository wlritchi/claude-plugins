---
description: Generate an opinionated, self-contained Dockerfile for a uv-based Python project
---

# Setup Dockerfile

Generate a ready-to-build Dockerfile using uv for dependency management. This command will produce a multi-stage Dockerfile example (minimal and production variants) and a brief explanation. The generated Dockerfile is self-contained with inline examples and does not reference any external files other than the project's pyproject.toml and uv.lock which you should have in your repo.

When you run this command, ask for the package entrypoint (module:callable or -m path), any runtime system packages (apt packages), and whether you want the minimal or production template. Example prompts:

- Entrypoint: your_package.entrypoint
- Runtime system deps (comma separated): libpq-dev, build-essential
- Template: production

Example output (production template):

# syntax=docker/dockerfile:1
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS build
WORKDIR /app
ARG VERSION=0.0.0
ENV SETUPTOOLS_SCM_PRETEND_VERSION=${VERSION} UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev
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



The command will replace placeholders like ${RUNTIME_APT} and the entrypoint with the values you provide when invoking the command.
