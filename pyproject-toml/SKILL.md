---
name: pyproject-toml
description: Create or modify pyproject.toml files for Python projects. Use when setting up a new Python project, configuring build tools, adding dependencies, or configuring development tools like black/isort/pytest.
allowed-tools: Read, Write, Edit, Glob
---

# pyproject.toml Configuration

## What This Skill Does

Creates and configures `pyproject.toml` files following PEP 621 standards. This is the modern standard for Python project configuration, replacing `setup.py` and `setup.cfg`.

## Complete Example

Reference example from this project (`./pyproject.toml`):

```toml
[build-system]
requires = ["setuptools", "setuptools-scm"]
build-backend = "setuptools.build_meta"

[project]
name = "home_financial_tools"
description = "Home Financial Tools"
requires-python = ">=3.12"
authors = [{ name = "Zhuo Yin", email = "zhuoyin@gmail.com" }]
dependencies = ["python-dateutil", "fpdf2", "fastapi", "uvicorn", "pyyaml", "pydantic", "bcrypt", "slowapi"]
dynamic = ["version"]

[project.scripts]
gen_invoice = "home_financial_tools.entry_points.invoice:main"

[tool.setuptools]
packages = ["home_financial_tools"]

[tool.setuptools.dynamic]
version = {attr = "home_financial_tools.__version__"}

[tool.black]
line-length = 120
skip-string-normalization = true
skip-magic-trailing-comma = true

[tool.isort]
profile = "black"
line_length = 120
```

## Section Reference

### 1. Build System (Required)

Specifies how to build the package:

```toml
[build-system]
requires = ["setuptools", "setuptools-scm"]
build-backend = "setuptools.build_meta"
```

| Backend | Use Case |
|---------|----------|
| `setuptools.build_meta` | Standard packages |
| `poetry.core.masonry.api` | Poetry projects |
| `hatchling` | Hatch projects |
| `flit_core.buildapi` | Simple pure-Python packages |

### 2. Project Metadata (Required)

Core project information per PEP 621:

```toml
[project]
name = "package-name"                    # Required: package name (use hyphens)
version = "1.0.0"                        # Version (or use dynamic)
description = "Short description"        # One-line description
requires-python = ">=3.12"               # Minimum Python version
authors = [
    { name = "Name", email = "email@example.com" }
]
dependencies = [                         # Runtime dependencies
    "requests>=2.28",
    "pydantic>=2.0",
]
dynamic = ["version"]                    # Fields computed at build time
```

### 3. Optional Dependencies

Group dependencies for optional features:

```toml
[project.optional-dependencies]
dev = ["pytest", "black", "isort", "mypy"]
docs = ["sphinx", "sphinx-rtd-theme"]
all = ["package[dev,docs]"]
```

Install with: `pip install package[dev]`

### 4. Entry Points / Scripts

Define CLI commands:

```toml
[project.scripts]
mycli = "mypackage.cli:main"              # Creates `mycli` command
```

### 5. Tool Configuration

#### Black (Code Formatter)

```toml
[tool.black]
line-length = 120
skip-string-normalization = true
skip-magic-trailing-comma = true
target-version = ["py312"]
exclude = '''
/(
    \.git
    | \.venv
    | build
    | dist
)/
'''
```

#### isort (Import Sorter)

```toml
[tool.isort]
profile = "black"
line_length = 120
known_first_party = ["mypackage"]
sections = ["FUTURE", "STDLIB", "THIRDPARTY", "FIRSTPARTY", "LOCALFOLDER"]
```

#### pytest (Testing)

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
addopts = "-v --tb=short"
```

#### mypy (Type Checking)

```toml
[tool.mypy]
python_version = "3.12"
strict = true
ignore_missing_imports = true
```

### 6. Setuptools Configuration

For setuptools-based projects:

```toml
[tool.setuptools]
packages = ["mypackage"]                 # Explicit package list
# OR use find:
package-dir = {"" = "src"}               # If using src layout

[tool.setuptools.packages.find]
where = ["src"]                          # For src/ layout
include = ["mypackage*"]

[tool.setuptools.dynamic]
version = {attr = "mypackage.__version__"}
readme = {file = ["README.md"]}
```

## Quick Start Templates

### Minimal Package

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "mypackage"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = []
```

### Application with CLI

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "myapp"
version = "1.0.0"
requires-python = ">=3.12"
dependencies = ["click", "rich"]

[project.scripts]
myapp = "myapp.cli:main"

[tool.setuptools]
packages = ["myapp"]
```

### Library with Dev Tools

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "mylib"
version = "1.0.0"
requires-python = ">=3.10"
dependencies = ["requests"]

[project.optional-dependencies]
dev = ["pytest", "black", "isort", "mypy"]

[tool.black]
line-length = 120
skip-string-normalization = true
skip-magic-trailing-comma = true

[tool.isort]
profile = "black"
line_length = 120

[tool.pytest.ini_options]
testpaths = ["tests"]
```

## Common Patterns

### Dynamic Version from `__version__`

1. Add to `pyproject.toml`:
```toml
[project]
dynamic = ["version"]

[tool.setuptools.dynamic]
version = {attr = "mypackage.__version__"}
```

2. Create `mypackage/__init__.py`:
```python
__version__ = "1.0.0"
```

### src/ Layout

```toml
[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]
```

## Validation

Check your pyproject.toml:

```bash
# Validate TOML syntax
python -c "import tomllib; tomllib.load(open('pyproject.toml', 'rb'))"

# Test build
pip install build
python -m build --sdist
```
