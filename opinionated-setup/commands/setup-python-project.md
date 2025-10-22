---
description: Create a new Python project skeleton with pyproject.toml, .pre-commit-config.yaml and step-by-step setup instructions
---

# Setup Python Project

Generate a complete, copy-pasteable project bootstrap for a modern Python project. This command outputs:

- A ready-to-use pyproject.toml (hatchling example) with sensible defaults
- A .pre-commit-config.yaml with pinned hook revisions
- Optional guidance for using hatch, uv, or a plain venv

When you run this command, it will ask:
- Project name (exampleproj)
- Author name and email
- Python minimum version (default: 3.12)
- Tools to include (choices: ruff, mypy, pytest, pre-commit, pyright)
- Packaging backend (hatchling or setuptools)

Example output (pyproject.toml):

[build-system]
requires = ["hatchling", "hatch-vcs"]
build-backend = "hatchling.build"

[project]
name = "exampleproj"
description = "Short description"
readme = "README.md"
authors = [ { name = "You", email = "you@example.com" } ]
requires-python = ">=3.12"
dynamic = ["version"]

[project.optional-dependencies]
dev = ["pre-commit", "ruff", "mypy", "pytest", "pyright"]

[tool.hatch]
version.source = "vcs"

[tool.ruff]
line-length = 88

[tool.ruff.format]
quote-style = "preserve"

[tool.ruff.lint]
ignore = ["E402", "E501", "C901"]
select = ["B", "C", "E", "F", "I", "W", "RUF100"]

[tool.mypy]
disallow_untyped_defs = true
explicit_package_bases = true
ignore_missing_imports = true

[tool.pytest.ini_options]
addopts = ["--import-mode=importlib"]


Example .pre-commit-config.yaml output:

repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-yaml
      - id: check-toml
      - id: check-json
      - id: check-merge-conflict

  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks:
      - id: yamllint

  - repo: https://github.com/codespell-project/codespell
    rev: v2.3.0
    hooks:
      - id: codespell
        args: [--quiet-level=2]

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: pytest
        language: system
        pass_filenames: false
        stages: [pre-push]


Setup instructions (copy/paste):

# Using a plain venv
python -m venv .venv
source .venv/bin/activate
python -m pip install -U pip setuptools wheel
pip install -e .[dev]
pip install pre-commit
pre-commit install
pre-commit run --all-files

# Using hatch (recommended)
pipx install hatch
hatch env create
hatch run python -m pytest
hatch run ruff check . --fix

# CI snippet (GitHub Actions) suggestion
name: CI
on: [push, pull_request]
jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: 3.12
      - name: Install deps
        run: |
          python -m pip install -U pip
          pip install hatch
          hatch env create
      - name: Run pre-commit
        run: pre-commit run --all-files
      - name: Run tests
        run: hatch run pytest


The command will substitute the project name, author, python version and selected tools into the templates when executed.
