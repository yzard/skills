---
name: docker-build
description: Create Docker build scripts for Python projects. Use when the user asks to create a build.sh script, dockerize an application, or set up Docker builds.
allowed-tools: Write, Read, Bash, Glob, Grep
---

# Docker Build Script Creation

## What This Skill Does

Creates Docker build infrastructure for Python projects:
- `build.sh` - Build script with date-based tagging
- `Dockerfile` - Alpine-based image with uv package manager
- `entrypoint.sh` - Supports PUID/PGID/UMASK/TZ for proper permissions

## Before Creating

**Detect project structure:**

1. Find the main Python package directory
2. Identify the entry point module (e.g., `main.py`, `app.py`, `server.py`)
3. Check for existing `Dockerfile`, `requirements.txt`, or `pyproject.toml`

```bash
# Find Python packages
find . -name "__init__.py" -maxdepth 2
# Find potential entry points
find . -name "main.py" -o -name "app.py" -o -name "server.py"
# Check existing files
ls -la Dockerfile requirements.txt pyproject.toml 2>/dev/null
```

## build.sh Template

Create `build.sh` in project root:

```bash
#!/bin/bash

# Get current date in YYYYMMDD format
TAG=$(date +%Y%m%d)
IMAGE_NAME="<username>/<project-name>"

echo "Building Docker image: ${IMAGE_NAME}:${TAG}..."

docker build -t "${IMAGE_NAME}:${TAG}" -t "${IMAGE_NAME}:latest" .

echo "Build complete: ${IMAGE_NAME}:${TAG}"

echo "Pushing Docker images to registry..."
docker push "${IMAGE_NAME}:${TAG}"
docker push "${IMAGE_NAME}:latest"

echo "Push complete: ${IMAGE_NAME}:${TAG} and ${IMAGE_NAME}:latest"
```

## entrypoint.sh Template

Create `entrypoint.sh` in project root. This script:
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

# Set umask
umask "$UMASK"

# Change ownership of data directory (app is owned by root, which is fine)
chown -R abc:abc /data 2>/dev/null || true

echo "Running as user abc ($(id abc))"

# Execute the application as the specified user
# Replace <package> and entry point as needed
exec su-exec abc:abc python -m <package>.main
```

## Dockerfile Template (Alpine + uv)

Create `Dockerfile` in project root. Key features:
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
COPY entrypoint.sh /entrypoint.sh
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

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PUID`   | 1000    | User ID for running the application |
| `PGID`   | 1000    | Group ID for running the application |
| `UMASK`  | 022     | File permission mask |
| `TZ`     | UTC     | Timezone (e.g., `America/New_York`, `Asia/Tokyo`) |

## Steps to Create

1. **Identify project name** - Use directory name or main package name
2. **Create build.sh** - Update `IMAGE_NAME` with your registry/project
3. **Create entrypoint.sh** - Update the `exec` line with your entry point
4. **Create Dockerfile** - Adjust port and paths as needed
5. **Create .dockerignore** - Exclude unnecessary files from build context
6. **Make scripts executable**:
   ```bash
   chmod +x build.sh entrypoint.sh
   ```

## Running

```bash
# Build the image
./build.sh

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

```yaml
version: "3.8"
services:
  app:
    image: <username>/<project-name>:latest
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=022
      - TZ=America/New_York
    volumes:
      - ./data:/data
    ports:
      - "8000:8000"
    restart: unless-stopped
```

## .dockerignore Template

Create `.dockerignore` in project root to exclude unnecessary files from the Docker build context:

```
# Python bytecode
**/__pycache__/
*.py[cod]
*$py.class

# Build artifacts
build/
dist/
sdist/
wheels/
eggs/
.eggs/
*.egg-info/
*.egg

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
Dockerfile
.dockerignore

# Pre-commit
.pre-commit-config.yaml

# Documentation
*.md
```

**Why use .dockerignore:**
- Speeds up builds by reducing context size
- Prevents unnecessary files from being copied to image
- Keeps images smaller and more secure
