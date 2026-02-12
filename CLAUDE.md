# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This repository provides examples for using and extending Posit's container images. It demonstrates two approaches:
- **bakery/**: Managing images with Posit's [Bakery tool](https://github.com/posit-dev/images-shared/tree/main/posit-bakery) (Jinja2-based templating system)
- **extending/**: Extending Posit's publicly available container images with customer-specific layers

> [!NOTE]
> These images are under active development and not yet officially supported by Posit. See [rstudio/rstudio-docker-products](https://github.com/rstudio/rstudio-docker-products) for officially supported images.

## Sibling repositories

This project is part of a multi-repo ecosystem for Posit container images. Sibling repos
are configured as additional directories (see `.claude/settings.json`). **Read the CLAUDE.md
in each affected sibling repo before making changes there.**

- `../images-shared/` - Posit Bakery CLI tool for building, testing, and managing container images. Jinja2 templates, macros, and shared build tooling.
- `../images/` - Meta repository with documentation, design principles, and links across all image repos.
- `../images-connect/` - Posit Connect images: `connect` (Standard/Minimal variants), `connect-content` (matrix of R x Python), `connect-content-init`.
- `../images-package-manager/` - Posit Package Manager image: `package-manager` (Standard/Minimal variants). Supports multi-platform builds (amd64/arm64).
- `../images-workbench/` - Posit Workbench images: `workbench` (Standard/Minimal variants), `workbench-session` (R x Python matrix), `workbench-session-init`.
- `../helm/` - Helm charts for Posit products: Connect, Workbench, Package Manager, and Chronicle.

### Worktrees for cross-repo changes

When making changes across repositories, use worktrees to isolate work from `main`. Multiple
sessions may be running concurrently, so never work directly on `main` in any repo.

- **Primary repo:** Use `EnterWorktree` with a descriptive name.
- **Sibling repos:** Create worktrees via `git worktree add` before making changes. Store
  them in `.claude/worktrees/<name>` within each repo (matching the `EnterWorktree` convention).

```bash
# Create a worktree in a sibling repo
git -C ../images-shared worktree add .claude/worktrees/<name> -b <branch-name>
```

Read and write files via the worktree path (e.g., `../images-shared/.claude/worktrees/<name>/`)
instead of the repo root. Clean up when finished:

```bash
git -C ../images-shared worktree remove .claude/worktrees/<name>
```

> **Note:** The `additionalDirectories` in `.claude/settings.json` point to the sibling repo
> roots, not to worktree paths. File reads and writes via those directories will access the
> repo root (typically on `main`). Always use the full worktree path when reading or writing
> files in a sibling worktree.

## Build commands

Build an extending example:
```bash
docker build -f extending/{example}/Containerfile -t {tag} extending/{example}/
```

CI runs on pull requests and builds all examples in `extending/` using Docker Buildx.

## Bakery tool architecture

Bakery uses Jinja2 templates to generate version-specific container build files.

### Directory structure for Bakery images
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

### Bakery template variables
- `{{ Image.Version }}` - Current image version string
- `{{ Path.Version }}` - Path to the version directory
- Import macros: `{%- import "apt.j2" as apt -%}`

### bakery.yaml structure
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

## Containerfile conventions

### Base image naming
- Format: `docker.io/posit/{product}:{version}-{variant}`
- Variants: `-min` (minimal), `-std` (standard), `-ubuntu-22.04-min`
- Examples:
  - `posit/workbench:2025.09.0-min`
  - `posit/connect:2025.07.0-ubuntu-22.04-min`
  - `posit/package-manager:{version}-ubuntu-22.04-min`

### Version pinning pattern
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

### Package installation patterns

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

### Cleanup requirements
- Always clean apt caches: `apt-get clean -yqq && rm -rf /var/lib/apt/lists/*`
- Delete R installer artifacts: `find . -type f -name '[rR]-{version}.*\.(deb|rpm)' -delete`
- Use `--no-cache-dir` with pip
- Use `clean = TRUE` with R `install.packages()`

## Key resources

- [Posit Public Package Manager](https://p3m.dev/) - Package repositories for R and Python
- [R installer script](https://rstd.io/r-install) - Automated R installation
- [Goss](https://github.com/goss-org/goss) - Container testing framework used by Bakery
- [GitHub Discussions](https://github.com/posit-dev/images/discussions) - Feedback and questions
