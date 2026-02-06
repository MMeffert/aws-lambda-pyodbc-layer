# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

AWS Lambda Layer builder for Python pyodbc with Microsoft ODBC drivers. Builds Docker containers on Amazon Linux 2/2023 that compile unixODBC from source, install Microsoft ODBC drivers, and package everything into Lambda-compatible zip layers.

## Build Commands

Build locally with Docker. Versions are defined in `versions.json`.

```bash
# AL2 (Python 3.10-3.11)
docker build -f Dockerfile-al2 \
  --build-arg PYTHON_VERSIONS="3.10,3.11" \
  --build-arg MSODBC_VERSION=18 \
  --build-arg UNIXODBC_VERSION=2.3.14 \
  -t pyodbc-layer:al2 .
CONTAINER_ID=$(docker create pyodbc-layer:al2)
docker cp $CONTAINER_ID:/opt/artifacts/. .
docker rm $CONTAINER_ID

# AL2023 (Python 3.12+)
docker build -f Dockerfile-al2023 \
  --build-arg PYTHON_VERSIONS="3.12,3.13,3.14" \
  --build-arg MSODBC_VERSION=18 \
  --build-arg UNIXODBC_VERSION=2.3.14 \
  -t pyodbc-layer:al2023 .
CONTAINER_ID=$(docker create pyodbc-layer:al2023)
docker cp $CONTAINER_ID:/opt/artifacts/. .
docker rm $CONTAINER_ID
```

Output: `pyodbc-layer-{python_version}-mssql{driver}-unixODBC{version}.zip`

## Testing

CI tests run against a real SQL Server 2022 container using the AWS Lambda Runtime Interface Emulator (RIE). No local test runner exists outside of the CI pipeline.

- `test/check_mssql.sh` — polls SQL Server container until ready (30 retries, 5s interval)
- `test/check_lambda.sh <msodbc_version>` — invokes Lambda via RIE and validates response
- `lambda/lambda_function.py` — test handler that connects to MSSQL and runs `SELECT @@VERSION`

## Architecture

### AL2 vs AL2023 Split

Python version determines the base image. This is driven by OpenSSL compatibility:
- **AL2** (`Dockerfile-al2`): Python <=3.11 — uses `openssl11-devel`, `yum`
- **AL2023** (`Dockerfile-al2023`): Python >=3.12 — uses native `openssl-devel`, `dnf`

The split is defined in `versions.json` under `python.al2` and `python.al2023`.

### Version Configuration

`versions.json` is the single source of truth for all version matrices. CI parses it in the `load-versions` job to generate build/test matrices. The `check-updates.yml` workflow runs weekly to scan for new Python runtimes and ODBC versions, creating PRs automatically.

### CI/CD Pipeline (`.github/workflows/build.yml`)

1. **load-versions** — parses `versions.json` into matrix outputs
2. **build** — Docker builds per MSODBC version; AL2 and AL2023 built sequentially in same job
3. **test** — matrix of (python_version x msodbc x unixodbc); spins up MSSQL 2022 + Lambda RIE
4. **release** — on `v*.*.*` tags: creates GitHub Release with all zip artifacts
5. **publish** — on tags: publishes Lambda Layer versions to Legacy (919311966619) and Production (344349181969) via OIDC cross-account role chaining through SharedServices (386930771048). Only MSODBC 18 is published.

### Layer Naming

- Zip files: `pyodbc-layer-{python_version}-mssql{driver}-unixODBC{version}.zip`
- Lambda layers: `pyodbc-layer-python{sanitized}-mssql{driver}` (dots removed from Python version: `3.10` -> `3-10`)

### Dockerfile Build Steps

Both Dockerfiles follow the same sequence:
1. Install build tools (gcc, make, autoconf, etc.)
2. Install pyenv for Python version management
3. Compile unixODBC from source with `/opt` prefix and UTF-8/UTF-16LE encoding
4. Install Microsoft ODBC Driver (17 or 18) from Microsoft's repo
5. Write `/opt/odbcinst.ini` and `/opt/odbc.ini` config files
6. For each Python version: install via pyenv, pip install pyodbc, zip `/opt` into artifact
