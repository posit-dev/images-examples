# Multiple images and versions example

This example demonstrates managing multiple image families with multiple versions each using Bakery. It builds multiple versions of two base images for different OS families, Ubuntu and Rocky Linux, showcasing Bakery's ability to handle diverse container configurations in a single project.

All command examples are expected to run with this example, `bakery/02-multiple-images-and-versions/`, as the working directory.

Bakery commands can also use the `--context PATH` option to specify the path to the example directory when running from a different location.

## Structure

```
02-multiple-images-and-versions/
├── bakery.yaml                    # Project configuration
├── ubuntu-base/
│   ├── template/                  # Jinja2 templates for Ubuntu
│   │   ├── Containerfile.jinja2
│   │   ├── deps/packages.txt.jinja2
│   │   └── test/goss.yaml.jinja2
│   ├── noble/                     # Generated files for Ubuntu 24.04
│   │   ├── Containerfile
│   │   ├── deps/packages.txt
│   │   └── test/goss.yaml
│   └── jammy/                     # Generated files for Ubuntu 22.04
│       ├── Containerfile
│       ├── deps/packages.txt
│       └── test/goss.yaml
└── rocky-base/
    ├── template/                  # Jinja2 templates for Rocky Linux
    │   ├── Containerfile.jinja2
    │   ├── deps/packages.txt.jinja2
    │   └── test/goss.yaml.jinja2
    ├── 9/                         # Generated files for Rocky Linux 9
    │   ├── Containerfile
    │   ├── deps/packages.txt
    │   └── test/goss.yaml
    └── 8/                         # Generated files for Rocky Linux 8
        ├── Containerfile
        ├── deps/packages.txt
        └── test/goss.yaml
```

## What this example builds

**ubuntu-base**:
- **Base**: Ubuntu 24.04 (noble) and Ubuntu 22.04 (jammy)
- **Packages**: `build-essential`, `git`
- **Versions**: 24.04, 22.04

**rocky-base**:
- **Base**: Rocky Linux 9 and Rocky Linux 8
- **Packages**: `binutils`, `cmake`, `gcc`, `make`, `git`, `strace`
- **Versions**: 9, 8

## Concepts

This example demonstrates several key Bakery features:

### Multiple image families

A single Bakery project can manage multiple distinct images. The [`bakery.yaml` file][BakeryConfiguration] defines two images (`ubuntu-base` and `rocky-base`), each with their own templates and version directories. This allows related images to share project configuration while maintaining separate build definitions.

While this example treats base OSes as a primary component, Bakery can also define them as a secondary dimension as demonstrated in [Example 4](../04-image-oses).

### Version matrices

Each image can have multiple versions. This example produces 4 container images total (2 images × 2 versions each):
- `ghcr.io/posit-dev/ubuntu-base:24.04`
- `ghcr.io/posit-dev/ubuntu-base:22.04`
- `ghcr.io/posit-dev/rocky-base:9`
- `ghcr.io/posit-dev/rocky-base:8`

### Subpath for custom directory names

Ubuntu versions use codenames (`noble`, `jammy`) instead of version numbers for directory names. The `subpath` field in `bakery.yaml` enables this:

```yaml
versions:
  - name: "24.04"
    subpath: "noble"    # Files generated to ubuntu-base/noble/
  - name: "22.04"
    subpath: "jammy"    # Files generated to ubuntu-base/jammy/
```

The `{{ Image.Version }}` variable still resolves to the version name (e.g., "24.04"), while `{{ Path.Version }}` uses the subpath (e.g., "ubuntu-base/noble").

### Package manager abstraction

Bakery provides macros for different package managers. Ubuntu uses `apt.j2` while Rocky Linux uses `dnf.j2`:

```jinja2
{# Ubuntu template #}
{%- import "apt.j2" as apt -%}
{{ apt.run_setup() }}
{{ apt.run_install(files=["/tmp/packages.txt"]) }}

{# Rocky template #}
{%- import "dnf.j2" as dnf -%}
{{ dnf.run_setup() }}
{{ dnf.run_install(files=["/tmp/packages.txt"]) }}
```

These macros abstract away the differences between package managers, handling cache updates, installation, and cleanup automatically.

### Latest tag management

The `latest: true` flag in `bakery.yaml` controls which version receives the `:latest` tag:

```yaml
images:
  - name: ubuntu-base
    versions:
      - name: "24.04"
        latest: true    # Tagged as ubuntu-base:24.04 AND ubuntu-base:latest
      - name: "22.04"   # Tagged only as ubuntu-base:22.04
```

## Creation of this example

The commands below were used to create this example from scratch.

```bash
# Initialize a new Bakery project
bakery create project

# Create the ubuntu-base image
bakery create image ubuntu-base

# Create the rocky-base image
bakery create image rocky-base

# Edit templates for each image to specify base images, package managers, and packages

# Add Ubuntu versions with subpaths for codename directories
bakery create version ubuntu-base 24.04 --subpath noble
bakery create version ubuntu-base 22.04 --subpath jammy

# Add Rocky Linux versions
bakery create version rocky-base 9
bakery create version rocky-base 8
```

## Building with Bakery CLI

Bakery manages the full lifecycle of rendering templates, building images, and running tests.

```bash
# Rerender templates to generate files for existing versions
bakery update files

# Build all images
bakery build

# Run tests for all images
bakery run dgoss
```

## Building directly with Docker

You can build images directly using Docker without Bakery. The build context must be the example directory (not the version directory) because the Containerfile references paths relative to it.

**Build ubuntu-base:24.04 (noble)**:
```bash
docker buildx build \
  --load \
  -f ubuntu-base/noble/Containerfile \
  -t ghcr.io/posit-dev/ubuntu-base:24.04 \
  -t ghcr.io/posit-dev/ubuntu-base:latest \
  .
```

**Build rocky-base:9**:
```bash
docker buildx build \
  --load \
  -f rocky-base/9/Containerfile \
  -t ghcr.io/posit-dev/rocky-base:9 \
  -t ghcr.io/posit-dev/rocky-base:latest \
  .
```

### Building using BuildKit Bake

As Bakery's name implies, we originally designed it as a wrapper around [Docker's BuildKit Bake tool](https://docs.docker.com/build/bake/). Bake allows for multiple concurrent builds from a shared configuration file. Bakery generates its own temporary configuration file for use with Bake named `.bakery-bake.json`. You can generate the contents of this file using `bakery build --plan` or retain the file for viewing after a build using `bakery build --no-clean`.

You can execute and load the plans generated by Bakery directly from Docker:

```bash
# Generate the Bake plan
bakery build --plan > docker-bake.json

# Run the Bake builds from the plan
docker buildx bake --file docker-bake.json --load
```

## Testing with dgoss

[Goss](https://github.com/goss-org/goss) is a serverspec-like tool for validating server configuration. [dgoss](https://github.com/goss-org/goss/tree/master/extras/dgoss) is a wrapper for testing Docker containers.

The goss.yaml files use Go templates to dynamically read the package list and verify each package is installed.

### Running tests manually

You can run tests without Bakery. The commands below would be functionally equivalent to running `bakery build` followed by `bakery run dgoss`. Always build and load the image locally, or pull it, before running tests.

**Test ubuntu-base:24.04**:
```bash
# Build the image first
docker buildx build \
  --load \
  -f ubuntu-base/noble/Containerfile \
  -t ghcr.io/posit-dev/ubuntu-base:24.04 \
  .

# Run dgoss
GOSS_FILES_PATH=ubuntu-base/noble/test \
dgoss run \
  -v "$(pwd)/ubuntu-base/noble:/tmp/version" \
  -v "$(pwd)/ubuntu-base:/tmp/image" \
  -v "$(pwd):/tmp/project" \
  -e IMAGE_VERSION=24.04 \
  -e IMAGE_VERSION_MOUNT=/tmp/version \
  -e IMAGE_MOUNT=/tmp/image \
  -e PROJECT_MOUNT=/tmp/project \
  --init \
  ghcr.io/posit-dev/ubuntu-base:24.04
```

**Test rocky-base:9**:
```bash
# Build the image first
docker buildx build \
  --load \
  -f rocky-base/9/Containerfile \
  -t ghcr.io/posit-dev/rocky-base:9 \
  .

# Run dgoss
GOSS_FILES_PATH=rocky-base/9/test \
dgoss run \
  -v "$(pwd)/rocky-base/9:/tmp/version" \
  -v "$(pwd)/rocky-base:/tmp/image" \
  -v "$(pwd):/tmp/project" \
  -e IMAGE_VERSION=9 \
  -e IMAGE_VERSION_MOUNT=/tmp/version \
  -e IMAGE_MOUNT=/tmp/image \
  -e PROJECT_MOUNT=/tmp/project \
  --init \
  ghcr.io/posit-dev/rocky-base:9
```

## Template variables

The Containerfile templates use these Bakery variables:

| Variable | Description | Example |
|:---------|:------------|:--------|
| `{{ Image.Version }}` | The version being built | `"24.04"`, `"9"` |
| `{{ Path.Version }}` | Path to the version directory | `"ubuntu-base/noble"`, `"rocky-base/9"` |

Note that when using `subpath`, `Image.Version` contains the version name while `Path.Version` reflects the subpath.

See [TEMPLATING.md](https://github.com/posit-dev/images-shared/blob/main/posit-bakery/TEMPLATING.md) for a list of available variables and macros.

## Bakery macros

This example uses Bakery's package manager macros to abstract differences between distributions:

**apt.j2** (for Debian/Ubuntu):

| Macro | Description |
|:------|:------------|
| `apt.run_setup()` | Installs required apt packages and sources |
| `apt.run_install(files=[...])` | Installs packages from files, handles cleanup |

**dnf.j2** (for RHEL/Rocky/Fedora):

| Macro | Description |
|:------|:------------|
| `dnf.run_setup()` | Installs required dnf packages and sources |
| `dnf.run_install(files=[...])` | Installs packages from files, handles cleanup |

These macros ensure consistent, best-practice package installation across different base images while keeping templates clean and readable.

[BakeryConfiguration]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#bakery-configuration
