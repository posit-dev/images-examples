# Image OSes Example

This example demonstrates how to build images for multiple operating systems combined with variants. It extends the variants concept from [example 03](../03-image-variants/) by adding multi-OS support, producing 6 container images from a single template set.

All command examples are expected to run with this example, `bakery/04-image-oses/`, as the working directory.

## Structure

```
04-image-oses/
├── bakery.yaml                                     # Project configuration with OS and variant support
└── example-image/
    ├── template/                                   # Jinja2 templates (source of truth)
    │   ├── Containerfile.ubuntu2404.jinja2
    │   ├── Containerfile.ubuntu2204.jinja2
    │   ├── Containerfile.rocky9.jinja2
    │   ├── deps/
    │   │   ├── ubuntu_packages.txt.jinja2
    │   │   ├── ubuntu_optional_packages.txt.jinja2
    │   │   ├── rocky_packages.txt.jinja2
    │   │   └── rocky_optional_packages.txt.jinja2
    │   └── test/goss.yaml.jinja2
    └── 1.0.0/                                      # Generated files for version 1.0.0
        ├── Containerfile.ubuntu2404.min
        ├── Containerfile.ubuntu2404.std
        ├── Containerfile.ubuntu2204.min
        ├── Containerfile.ubuntu2204.std
        ├── Containerfile.rocky9.min
        ├── Containerfile.rocky9.std
        ├── deps/
        │   ├── ubuntu_packages.txt
        │   ├── ubuntu_optional_packages.txt
        │   ├── rocky_packages.txt
        │   └── rocky_optional_packages.txt
        └── test/goss.yaml
```

## What This Example Builds

This example creates a matrix of images across operating systems and variants:

**Operating Systems**:
- Ubuntu 24.04 (primary)
- Ubuntu 22.04
- Rocky Linux 9

**Variants**:
- Standard: Core + optional packages
- Minimal: Core packages only

**Version**: 1.0.0

Produces 6 images with tags like:
- `example-image:1.0.0-ubuntu-24.04-standard`
- `example-image:1.0.0-ubuntu-24.04-minimal`
- `example-image:1.0.0-ubuntu-22.04-standard`
- `example-image:1.0.0-ubuntu-22.04-minimal`
- `example-image:1.0.0-rocky-9-standard`
- `example-image:1.0.0-rocky-9-minimal`

## Concepts

This example demonstrates several key Bakery features for multi-OS image management:

### OS Configuration in bakery.yaml

[Operating systems][ImageVersionOS] are defined within each version's configuration:

```yaml
images:
  - name: example-image
    variants:
      - name: Standard
        extension: std
        primary: true
      - name: Minimal
        extension: min
    versions:
      - name: 1.0.0
        latest: true
        os:
          - name: "Ubuntu 24.04"
            primary: true
          - name: "Ubuntu 22.04"
          - name: "Rocky 9"
```

The `primary: true` flag on Ubuntu 24.04 marks it as the default OS for version tags.

### OS-Specific Templates

Each operating system requires its own Containerfile template named with a condensed OS identifier suffix:

- `Containerfile.ubuntu2404.jinja2` - Ubuntu 24.04 template
- `Containerfile.ubuntu2204.jinja2` - Ubuntu 22.04 template
- `Containerfile.rocky9.jinja2` - Rocky Linux 9 template

The extension defaults to an all-lowercase condensed version of the OS name (e.g., "Ubuntu 24.04" → "ubuntu2404"). You can override this with the `extension` field in the OS configuration if needed.

Each template uses the appropriate base image and package manager macro:

```jinja2
{# Ubuntu 24.04 template #}
{%- import "apt.j2" as apt -%}
FROM docker.io/library/ubuntu:24.04
...
{{ apt.run_install(files=package_files) }}
```

```jinja2
{# Rocky 9 template #}
{%- import "dnf.j2" as dnf -%}
FROM docker.io/library/rockylinux:9
...
{{ dnf.run_install(files=package_files) }}
```

### OS-Specific Package Lists

Package files are named with an OS prefix and referenced dynamically using the `{{ Image.OS.Name }}` template variable:

- `ubuntu_packages.txt` / `ubuntu_optional_packages.txt`
- `rocky_packages.txt` / `rocky_optional_packages.txt`

The template dynamically selects the correct package file:

```jinja2
COPY {{ Path.Version }}/deps/{{ Image.OS.Name }}_packages.txt /tmp/packages.txt
{% if Image.Variant != "Minimal" -%}
COPY {{ Path.Version }}/deps/{{ Image.OS.Name }}_optional_packages.txt /tmp/optional_packages.txt
{% endif -%}
```

This pattern handles package name differences between distributions:

| Ubuntu | Rocky |
|--------|-------|
| `build-essential` | `binutils`, `cmake`, `gcc`, `make` |
| `libodbc2` | `unixODBC` |
| `libpq-dev` | `libpq-devel` |

### Package Manager Abstraction

Bakery provides macros that abstract package manager differences:

- **apt.j2** (Debian/Ubuntu): Uses `apt-get` with automatic cache cleanup
- **dnf.j2** (RHEL/Rocky/Fedora): Uses `dnf` with automatic cache cleanup

Both provide a consistent `run_install(files=[...])` interface, keeping templates clean while generating OS-appropriate commands.

### Combined with Variants

The same variant logic from [example 03](../03-image-variants/) applies here:

- **Minimal variant**: Installs only base packages (`packages.txt`)
- **Standard variant**: Installs base + optional packages (`optional_packages.txt`)

The combination of 3 OSes × 2 variants × 1 version produces 6 unique Containerfiles.

### OS-Agnostic Testing

A single `goss.yaml.jinja2` template works for all OSes by using environment variables to locate the correct package files:

```yaml
package:
  {{ $package_file := printf "/tmp/version/deps/%s_packages.txt" .Env.IMAGE_OS_NAME }}
  {{ $package_list := readFile $package_file | splitList "\n" }}
  {{- range $package_list }}
  {{.}}:
    installed: true
  {{end}}
```

The `IMAGE_OS_NAME` environment variable ("ubuntu" or "rocky") directs the test to the appropriate package list, while `IMAGE_VARIANT` controls whether optional packages are expected to be installed.

## Creation of this Example

```bash
# Initialize a new Bakery project
bakery create project

# Create the image
bakery create image example-image

# Edit bakery.yaml to add variants and os blocks

# Create OS-specific templates:
# - Containerfile.ubuntu2404.jinja2
# - Containerfile.ubuntu2204.jinja2
# - Containerfile.rocky9.jinja2

# Create OS-specific package files in template/deps/

# Add the first version
bakery create version example-image 1.0.0
```

## Building with Bakery CLI

Bakery manages the full lifecycle of rendering templates, building images, and running tests.

```bash
# Rerender templates to generate files for existing versions
bakery update files

# Build all OS/variant combinations
bakery build

# Run tests for all images
bakery run dgoss
```

## Building Directly with Docker

You can build each OS/variant combination directly using Docker. The build context must be the example directory because the Containerfile references paths relative to it.

### Build Ubuntu 24.04 Standard

```bash
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.ubuntu2404.std \
  -t ghcr.io/posit-dev/example-image:1.0.0-ubuntu-24.04-standard \
  .
```

### Build Ubuntu 22.04 Minimal

```bash
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.ubuntu2204.min \
  -t ghcr.io/posit-dev/example-image:1.0.0-ubuntu-22.04-minimal \
  .
```

### Build Rocky 9 Standard

```bash
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.rocky9.std \
  -t ghcr.io/posit-dev/example-image:1.0.0-rocky-9-standard \
  -t ghcr.io/posit-dev/example-image:1.0.0-rocky-9 \
  .
```

## Testing with dgoss

[Goss](https://github.com/goss-org/goss) is a serverspec-like tool for validating server configuration. [dgoss](https://github.com/goss-org/goss/tree/master/extras/dgoss) is a wrapper for testing Docker containers.

The goss.yaml template uses environment variables to handle different OS/variant combinations:
- `IMAGE_OS_NAME`: OS identifier ("ubuntu" or "rocky") for locating package files
- `IMAGE_VARIANT`: Variant name ("Minimal" or "Standard") for determining expected packages

### Running Tests Manually

You can run tests without Bakery. Set both `IMAGE_OS_NAME` and `IMAGE_VARIANT` correctly.

#### Test Ubuntu 24.04 Standard

```bash
# Build the image first
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.ubuntu2404.std \
  -t ghcr.io/posit-dev/example-image:1.0.0-ubuntu-24.04-standard \
  .

# Run dgoss
GOSS_FILES_PATH=example-image/1.0.0/test \
dgoss run \
  -v "$(pwd)/example-image/1.0.0:/tmp/version" \
  -v "$(pwd)/example-image:/tmp/image" \
  -v "$(pwd):/tmp/project" \
  -e IMAGE_VERSION=1.0.0 \
  -e IMAGE_VARIANT=Standard \
  -e IMAGE_OS_NAME=ubuntu \
  -e IMAGE_VERSION_MOUNT=/tmp/version \
  -e IMAGE_MOUNT=/tmp/image \
  -e PROJECT_MOUNT=/tmp/project \
  --init \
  ghcr.io/posit-dev/example-image:1.0.0-ubuntu-24.04-standard
```

#### Test Rocky 9 Minimal

```bash
# Build the image first
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.rocky9.min \
  -t ghcr.io/posit-dev/example-image:1.0.0-rocky-9-minimal \
  .

# Run dgoss
GOSS_FILES_PATH=example-image/1.0.0/test \
dgoss run \
  -v "$(pwd)/example-image/1.0.0:/tmp/version" \
  -v "$(pwd)/example-image:/tmp/image" \
  -v "$(pwd):/tmp/project" \
  -e IMAGE_VERSION=1.0.0 \
  -e IMAGE_VARIANT=Minimal \
  -e IMAGE_OS_NAME=rocky \
  -e IMAGE_VERSION_MOUNT=/tmp/version \
  -e IMAGE_MOUNT=/tmp/image \
  -e PROJECT_MOUNT=/tmp/project \
  --init \
  ghcr.io/posit-dev/example-image:1.0.0-rocky-9-minimal
```

## Template Variables

The Containerfile templates use these Bakery variables:

| Variable | Description | Example |
|:---------|:------------|:--------|
| `{{ Image.Version }}` | The version being built | `"1.0.0"` |
| `{{ Image.Variant }}` | The variant being built | `"Minimal"`, `"Standard"` |
| `{{ Image.OS.Name }}` | The OS identifier | `"ubuntu"`, `"rocky"` |
| `{{ Path.Version }}` | Path to the version directory | `"example-image/1.0.0"` |

See [TEMPLATING.md](https://github.com/posit-dev/images-shared/blob/main/posit-bakery/TEMPLATING.md) for a list of available variables and macros.

[ImageVersionOS]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#imageversionos
