# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository provides examples for using and extending Posit's container images. It demonstrates two approaches:
- **bakery/**: Managing images with Posit's [Bakery tool](https://github.com/posit-dev/images-shared/tree/main/posit-bakery) (Jinja2-based templating system)
- **extending/**: Extending Posit's publicly available container images with customer-specific layers

> **Note**: These images are under active development and not yet officially supported by Posit. See [rstudio/rstudio-docker-products](https://github.com/rstudio/rstudio-docker-products) for officially supported images.

## Build Commands

Build an extending example:
```bash
docker build -f extending/{example}/Containerfile -t {tag} extending/{example}/
```

CI runs on pull requests and builds all examples in `extending/` using Docker Buildx.

## Bakery Tool Architecture

Bakery uses Jinja2 templates to generate version-specific container build files.

### Directory Structure for Bakery Images
```
bakery/{example}/
├── bakery.yaml                    # Repository config (registries, images, versions)
└── {image-name}/
    ├── template/                  # Jinja2 source templates
    │   ├── Containerfile.jinja2
    │   ├── deps/packages.txt.jinja2
    │   └── test/goss.yaml.jinja2
    └── {version}/                 # Generated files (rendered from templates)
        ├── Containerfile
        ├── deps/packages.txt
        └── test/goss.yaml
```

### Bakery Template Variables
- `{{ Image.Version }}` - Current image version string
- `{{ Path.Version }}` - Path to the version directory
- Import macros: `{%- import "apt.j2" as apt -%}`

### bakery.yaml Structure
```yaml
repository:
  url: "github.com/posit-dev/images-examples"
  vendor: "Posit Software, PBC"
  maintainer: "Posit Docker Team <docker@posit.co>"
registries:
  - host: "ghcr.io"
    namespace: "posit-dev"
images:
  - name: example-image
    versions:
      - name: 1.0.0
        latest: true
```

## Containerfile Conventions

### Base Image Naming
- Format: `docker.io/posit/{product}:{version}-{variant}`
- Variants: `-min` (minimal), `-std` (standard), `-ubuntu-22.04-min`
- Examples:
  - `posit/workbench:2025.09.0-min`
  - `posit/connect:2025.07.0-ubuntu-22.04-min`
  - `posit/package-manager:{version}-ubuntu-22.04-min`

### Version Pinning Pattern
Always declare product versions as ARGs at the top:
```dockerfile
ARG PWB_VERSION="2025.09.0"
FROM docker.io/posit/workbench:${PWB_VERSION}-min
```

### Installing Python (via uv)
Multi-stage build using [uv](https://github.com/astral-sh/uv):
```dockerfile
FROM ghcr.io/astral-sh/uv:bookworm-slim AS python-builder
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
ENV UV_PYTHON_INSTALL_DIR=/opt/python
ENV UV_PYTHON_PREFERENCE=only-managed
RUN uv python install 3.13.7 3.12.11

FROM docker.io/posit/workbench:${PWB_VERSION}-min
COPY --from=python-builder /opt/python /opt/python
```

Python installs to `/opt/python/cpython-{version}-linux-x86_64-gnu/`.

### Installing R
```dockerfile
RUN RUN_UNATTENDED=1 R_VERSION=4.5.1 bash -c "$(curl -fsSL https://rstd.io/r-install)" && \
    find . -type f -name '[rR]-4.5.1.*\.(deb|rpm)' -delete
```

R installs to `/opt/R/{version}/`.

### Package Installation Patterns

Python packages:
```dockerfile
COPY requirements.txt /tmp/requirements.txt
RUN /opt/python/cpython-{version}-linux-x86_64-gnu/bin/pip install \
    --no-cache-dir --upgrade --break-system-packages -r /tmp/requirements.txt
```

R packages:
```dockerfile
COPY packages.txt /tmp/packages.txt
RUN /opt/R/{version}/bin/R --vanilla -e \
    'install.packages(readLines("/tmp/packages.txt"), repos="https://p3m.dev/cran/__linux__/jammy/latest", clean = TRUE)'
```

System packages:
```dockerfile
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update -yqq && \
    apt-get install -yqq --no-install-recommends {packages} && \
    apt-get clean -yqq && rm -rf /var/lib/apt/lists/*
```

### Cleanup Requirements
- Always clean apt caches: `apt-get clean -yqq && rm -rf /var/lib/apt/lists/*`
- Delete R installer artifacts: `find . -type f -name '[rR]-{version}.*\.(deb|rpm)' -delete`
- Use `--no-cache-dir` with pip
- Use `clean = TRUE` with R `install.packages()`

## Key Resources

- [Posit Public Package Manager](https://p3m.dev/) - Package repositories for R and Python
- [R installer script](https://rstd.io/r-install) - Automated R installation
- [Goss](https://github.com/goss-org/goss) - Container testing framework used by Bakery
- [GitHub Discussions](https://github.com/posit-dev/images/discussions) - Feedback and questions