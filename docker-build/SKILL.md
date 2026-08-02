---
name: docker-build
description: Create Docker build infrastructure for Python and/or Rust projects - Dockerfile, entrypoint.sh, and a build_docker.sh script. Use when the user asks to create a build script, dockerize an application, set up Docker builds, or add a Rust binary alongside a Python app.
allowed-tools: Write, Read, Bash, Glob, Grep
---

# Docker Build Script Creation

## What This Skill Does

Creates Docker build infrastructure for Python and/or Rust projects:
- `build_docker.sh` - Build script with date-based tagging, at the git root
- `Dockerfile` - Alpine-based multi-stage image; supports Python (uv), Rust, or both
- `entrypoint.sh` - Supports PUID/PGID/UMASK/TZ for proper permissions

This skill owns the **contents** of those files. Their **locations** are owned by the
`project-structure` skill, Rule 6 — summarized below, with the path consequences
(build context, `.dockerignore` naming, compose paths) documented there.

## File Locations

All Docker files live in `<git root>/docker/`. The single exception is
`build_docker.sh`, which sits at the git root and cds into `docker/` to build.

```
<git root>/
├── build_docker.sh              # the only Docker-related file at the root
└── docker/
    ├── Dockerfile
    ├── Dockerfile.dockerignore  # NOT .dockerignore — see project-structure Rule 6
    ├── docker-compose.yml
    └── entrypoint.sh
```

The build context is the **git root**, not `docker/`, so the Dockerfile can copy
`src/`. Every `COPY` path in the templates below is relative to the git root.

## Before Creating

**Detect project structure:**

1. Find the main Python package directory and/or Rust workspace/crate
2. Identify the Python entry point module (e.g., `main.py`, `app.py`, `server.py`)
3. Identify the Rust binary name from `Cargo.toml` (`[[bin]]` or `[package].name`)
4. Check for existing `Dockerfile`, `requirements.txt`, `pyproject.toml`, or `Cargo.toml`

```bash
# Find Python packages
find . -name "__init__.py" -maxdepth 2
# Find potential Python entry points
find . -name "main.py" -o -name "app.py" -o -name "server.py"
# Find Rust crates
find . -name "Cargo.toml" -maxdepth 3
# Check existing files
ls -la docker/ build_docker.sh requirements.txt pyproject.toml Cargo.toml 2>/dev/null
```

## build_docker.sh Template

Create `build_docker.sh` at the **git root**:

```bash
#!/bin/bash
set -euo pipefail

# Enter docker/ so the script works from any working directory
cd "$(dirname "$0")/docker"

# Get current date in YYYYMMDD format
TAG=$(date +%Y%m%d)
IMAGE_NAME="<username>/<project-name>"

echo "Building Docker image: ${IMAGE_NAME}:${TAG}..."

# -f Dockerfile with `..` as context: the build context is the git root
docker build -f Dockerfile -t "${IMAGE_NAME}:${TAG}" -t "${IMAGE_NAME}:latest" ..

echo "Build complete: ${IMAGE_NAME}:${TAG}"

echo "Pushing Docker images to registry..."
docker push "${IMAGE_NAME}:${TAG}"
docker push "${IMAGE_NAME}:latest"

echo "Push complete: ${IMAGE_NAME}:${TAG} and ${IMAGE_NAME}:latest"
```

## entrypoint.sh Template

Create `docker/entrypoint.sh`. This script:
- Creates user/group with specified PUID/PGID
- Sets UMASK for file permissions
- Sets timezone via TZ
- Runs the application as non-root user

```bash
#!/bin/sh

# Set default values if not provided
PUID=${PUID:-1000}
PGID=${PGID:-1000}
UMASK=${UMASK:-022}

echo "Starting with PUID=$PUID, PGID=$PGID, UMASK=$UMASK"

# Set timezone if TZ is provided
if [ -n "$TZ" ]; then
    echo "Setting timezone to $TZ"
    cp /usr/share/zoneinfo/$TZ /etc/localtime 2>/dev/null || true
    echo "$TZ" > /etc/timezone 2>/dev/null || true
fi

# Create group if it doesn't exist
if ! getent group abc > /dev/null 2>&1; then
    addgroup -g "$PGID" abc
fi

# Create user if it doesn't exist
if ! id abc > /dev/null 2>&1; then
    adduser -D -u "$PUID" -G abc -h /app abc
fi

# Change ownership of data directory (app is owned by root, which is fine)
chown -R abc:abc /data 2>/dev/null || true

echo "Running as user abc ($(id abc))"

# Execute the application as the specified user
# Replace <package> and entry point as needed
exec su-exec abc:abc sh -c "umask '$UMASK' && exec python -m <package>.main"
```

## Dockerfile Template (Alpine + uv)

Create `docker/Dockerfile`. Key features:
- **Alpine Linux** base for smaller image size
- **uv** package manager installed from Alpine repos
- **su-exec** for dropping privileges
- **PUID/PGID/UMASK/TZ** environment variables

```dockerfile
FROM python:3.12-alpine

WORKDIR /app

# Install system dependencies
# - build-base: for compiling Python packages with C extensions
# - su-exec: for dropping privileges to specified user
# - shadow: for user/group management
# - uv: fast Python package manager
# - tzdata: for timezone support
RUN apk add --no-cache \
    build-base \
    su-exec \
    shadow \
    uv \
    tzdata

# Copy project files
COPY . .

# Install Python dependencies using uv (system-wide, no venv)
RUN uv pip install --system --no-cache .

# Create data directory for volume mounting
RUN mkdir -p /data

# Copy entrypoint script
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# Set default environment variables
ENV PUID=1000 \
    PGID=1000 \
    UMASK=022 \
    TZ=UTC \
    PYTHONPATH=/app

# Expose port (adjust as needed)
EXPOSE 8000

ENTRYPOINT ["/entrypoint.sh"]
```

## Alternative: requirements.txt Installation

If project uses `requirements.txt` instead of `pyproject.toml`:

```dockerfile
# Replace the uv pip install line with:
COPY requirements.txt .
RUN uv pip install --system --no-cache -r requirements.txt

# Then copy application code
COPY <package>/ ./<package>/
```

## Dockerfile Template (Rust-only, Alpine)

For projects that are purely Rust with no Python runtime:

```dockerfile
# ── Stage 1: Build Rust binary ────────────────────────────────────────────────
FROM rust:1.84-alpine AS rust-builder

WORKDIR /app

# musl-dev + build-base for static linking on Alpine
# openssl-dev / openssl-libs-static if the crate links OpenSSL
RUN apk add --no-cache build-base musl-dev pkgconfig openssl-dev openssl-libs-static

COPY <rust-crate>/ /app/<rust-crate>/

RUN cargo build --release --manifest-path /app/<rust-crate>/Cargo.toml

# ── Stage 2: Runtime image ────────────────────────────────────────────────────
FROM alpine:3.21

WORKDIR /app

# ca-certificates: TLS root CAs
# su-exec: privilege-drop helper
# tzdata: timezone support
RUN apk add --no-cache ca-certificates su-exec tzdata

COPY --from=rust-builder /app/<rust-crate>/target/release/<binary-name> /app/<binary-name>
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

RUN mkdir -p /data

VOLUME ["/data"]

ENV PUID=1000 \
    PGID=1000 \
    UMASK=022 \
    TZ=UTC

EXPOSE 8000

ENTRYPOINT ["/entrypoint.sh"]
```

`entrypoint.sh` exec line for Rust-only:
```sh
exec su-exec abc:abc sh -c "umask '$UMASK' && exec /app/<binary-name>"
```

## Dockerfile Template (Python + Rust, Alpine)

For projects that ship **both** a Python application and a Rust binary in a single image:

```dockerfile
# ── Stage 1: Build Rust binary ────────────────────────────────────────────────
FROM rust:1.84-alpine AS rust-builder

WORKDIR /app

RUN apk add --no-cache build-base musl-dev pkgconfig openssl-dev openssl-libs-static

COPY <rust-crate>/ /app/<rust-crate>/

RUN cargo build --release --manifest-path /app/<rust-crate>/Cargo.toml

# ── Stage 2: Runtime image ────────────────────────────────────────────────────
FROM python:3.12-alpine

WORKDIR /app

# su-exec: privilege-drop; tzdata: timezone; uv: fast Python package manager
# openssl: required by the Rust binary at runtime (omit if binary is statically linked)
RUN apk add --no-cache \
    build-base \
    ca-certificates \
    openssl \
    su-exec \
    tzdata \
    uv

# Copy Rust binary
COPY --from=rust-builder /app/<rust-crate>/target/release/<binary-name> /app/<binary-name>

# Install Python dependencies
COPY requirements.txt .
RUN uv pip install --system --no-cache -r requirements.txt

# Copy Python application code and supporting files
COPY <python-package>/ ./<python-package>/
COPY configs/ ./configs/

COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

RUN mkdir -p /data

VOLUME ["/data"]

# PUID/PGID/UMASK/TZ are the standard permission-control variables
ENV PUID=1000 \
    PGID=1000 \
    UMASK=022 \
    TZ=UTC \
    PYTHONPATH=/app

EXPOSE 8000

ENTRYPOINT ["/entrypoint.sh"]
```

`entrypoint.sh` exec line when both run together (e.g. Rust binary is the main process):
```sh
exec su-exec abc:abc sh -c "umask '$UMASK' && exec /app/<binary-name>"
```

Or if Python is the main process and Rust binary is a sidecar started first:
```sh
/app/<binary-name> &
exec su-exec abc:abc sh -c "umask '$UMASK' && exec python -m <python-package>.main"
```

### Static vs Dynamic Rust binary on Alpine

Alpine uses **musl libc**. By default `cargo build` produces a dynamically linked binary that links musl. If linking against OpenSSL, either:

1. **Static OpenSSL** (preferred) — add `openssl-libs-static` in the builder and set:
   ```
   RUSTFLAGS="-C target-feature=+crt-static"
   ```
   The resulting binary needs **no OpenSSL** at runtime.

2. **Dynamic OpenSSL** — install `openssl` in the runtime image (as shown above).

Key rule: if the builder used `openssl-libs-static`, omit `openssl` from the runtime `apk add`. If not, include it.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PUID`   | 1000    | User ID for running the application |
| `PGID`   | 1000    | Group ID for running the application |
| `UMASK`  | 022     | File permission mask |
| `TZ`     | UTC     | Timezone (e.g., `America/New_York`, `Asia/Tokyo`) |

## Steps to Create

1. **Identify project name** - Use directory name or main package name
2. **Determine stack** - Python-only, Rust-only, or Python + Rust
3. **Create the `docker/` directory** at the git root
4. **Create build_docker.sh at the git root** - Update `IMAGE_NAME` with your registry/project; keep the `cd` and the `..` context
5. **Create docker/entrypoint.sh** - Update the `exec` line with your entry point; use `su-exec` for privilege drop
6. **Choose Dockerfile template** → `docker/Dockerfile` - Python-only, Rust-only, or combined; adjust Rust crate paths, binary name, Python package paths, and port. All `COPY` paths are relative to the git root
7. **Create docker/Dockerfile.dockerignore** - Exclude unnecessary files from build context
8. **Make scripts executable**:
   ```bash
   chmod +x build_docker.sh docker/entrypoint.sh
   ```

## Running

```bash
# Build the image (works from any directory)
./build_docker.sh

# Run with custom user/group/timezone
docker run -d \
    -e PUID=1000 \
    -e PGID=1000 \
    -e UMASK=022 \
    -e TZ=America/New_York \
    -v /path/to/data:/data \
    -p 8000:8000 \
    <username>/<project-name>:latest
```

## Docker Compose Example

Create `docker/docker-compose.yml`. Compose resolves relative paths against the
**compose file's own directory**, so the build context and every volume path must
step back up to the git root with `..`:

```yaml
services:
  app:
    image: <username>/<project-name>:latest
    build:
      context: ..
      dockerfile: docker/Dockerfile
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=022
      - TZ=America/New_York
    volumes:
      - ../playground/data:/data
    ports:
      - "8000:8000"
    restart: unless-stopped
```

## Dockerfile.dockerignore Template

Create `docker/Dockerfile.dockerignore` to exclude unnecessary files from the build context.

**The name matters.** Docker resolves `.dockerignore` relative to the build context
(the git root), so a file named `docker/.dockerignore` would be silently ignored with
no warning. BuildKit checks `<dockerfile-name>.dockerignore` beside the Dockerfile
first, so `docker/Dockerfile.dockerignore` is the form that keeps the file in `docker/`
and actually takes effect. Requires BuildKit (the default in current Docker).

```
# Python bytecode
**/__pycache__/
*.py[cod]
*$py.class

# Python build artifacts
build/
dist/
sdist/
wheels/
eggs/
.eggs/
*.egg-info/
*.egg

# Rust build artifacts
**/target/
**/*.rs.bk
.cargo/

# IDE and editor
.idea/
.vscode/

# Virtual environments
.venv/
venv*/

# Cache
.mypy_cache/

# Development files
playground/
*.log

# Database files
*.sqlite
*.sqlite-journal
*.db

# Git
.git/
.gitignore

# Docker
docker/

# Pre-commit
.pre-commit-config.yaml

# Documentation
*.md
```

**Why use .dockerignore:**
- Speeds up builds by reducing context size
- Prevents unnecessary files from being copied to image
- Keeps images smaller and more secure
