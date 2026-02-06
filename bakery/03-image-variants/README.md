# Image Variants Example

This example demonstrates how to build multiple variants of the same image from a single template. It creates both minimal and standard variants of an Ubuntu 22.04 image, each with different package sets.

All command examples are expected to run with this example, `bakery/03-image-variants/`, as the working directory.

## Structure

```
03-image-variants/
├── bakery.yaml                              # Project configuration with variants
└── example-image/
    ├── template/                            # Jinja2 templates (source of truth)
    │   ├── Containerfile.jinja2
    │   ├── deps/packages.txt.jinja2
    │   ├── deps/optional_packages.txt.jinja2
    │   └── test/goss.yaml.jinja2
    └── 1.0.0/                               # Generated files for version 1.0.0
        ├── Containerfile.min                # Minimal variant
        ├── Containerfile.std                # Standard variant
        ├── deps/packages.txt
        ├── deps/optional_packages.txt
        └── test/goss.yaml
```

## What This Example Builds

- **Base**: Ubuntu 22.04
- **Variants**:
  - **Minimal**: Core packages only (`build-essential`, `ca-certificates`, `curl`, `git`)
  - **Standard**: Core + optional packages (`libodbc2`, `libpq-dev`)
- **Version**: 1.0.0
- **Produces**: `example-image:1.0.0-minimal` and `example-image:1.0.0-standard`

## Concepts

### Variant Configuration in bakery.yaml

Variants are defined in the image configuration with `name`, `extension`, and `primary` fields:

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
```

- **name**: Human-readable variant name, accessible as `{{ Image.Variant }}` in templates
- **extension**: Suffix appended to generated filenames (e.g., `Containerfile.std`)
- **primary**: Marks the default variant; the primary variant's extension is also used for version tags (e.g., `example-image:1.0.0-standard`)

### Conditional Template Logic

Templates use the `{{ Image.Variant }}` variable to include variant-specific content:

```jinja2
COPY {{ Path.Version }}/deps/packages.txt /tmp/packages.txt
{% if Image.Variant != "Minimal" -%}
COPY {{ Path.Version }}/deps/optional_packages.txt /tmp/optional_packages.txt
{% set package_files = ["/tmp/packages.txt", "/tmp/optional_packages.txt"] -%}
{% else -%}
{% set package_files = ["/tmp/packages.txt"] -%}
{% endif -%}
{{ apt.run_install(files=package_files) }}
```

This pattern allows a single template to generate different Containerfiles based on the variant being rendered.

### Variant-Specific Output Files

Bakery generates separate Containerfiles for each variant, distinguished by their extensions:

- `Containerfile.min` - Standalone build file for the Minimal variant
- `Containerfile.std` - Standalone build file for the Standard variant

Each generated file is complete and can be built independently.

### Package Separation Pattern

This example separates packages into two files:
- `packages.txt` - Core packages installed in all variants
- `optional_packages.txt` - Additional packages installed only in the Standard variant

This pattern keeps package lists maintainable and makes variant differences explicit.

## Creation of this Example

```bash
# Initialize a new Bakery project
bakery create project

# Create the image (edit bakery.yaml to add variants configuration)
bakery create image example-image

# Edit templates to add conditional variant logic

# Add the first version
bakery create version example-image 1.0.0
```

## Building with Bakery CLI

Bakery manages the full lifecycle of rendering templates, building images, and running tests.

```bash
# Rerender templates to generate files for existing versions
bakery update files

# Build all variants
bakery build

# Run tests for all variants
bakery run dgoss
```

## Building Directly with Docker

You can build each variant directly using Docker. The build context must be the example directory because the Containerfile references paths relative to it.

### Build the Minimal Variant

```bash
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.min \
  -t ghcr.io/posit-dev/example-image:1.0.0-minimal \
  -t ghcr.io/posit-dev/example-image:minimal \
  .
```

### Build the Standard Variant

```bash
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.std \
  -t ghcr.io/posit-dev/example-image:1.0.0-standard \
  -t ghcr.io/posit-dev/example-image:standard \
  -t ghcr.io/posit-dev/example-image:latest \
  .
```

## Testing with dgoss

[Goss](https://github.com/goss-org/goss) is a serverspec-like tool for validating server configuration. [dgoss](https://github.com/goss-org/goss/tree/master/extras/dgoss) is a wrapper for testing Docker containers.

The goss.yaml template uses the `IMAGE_VARIANT` environment variable to conditionally check whether optional packages should be installed:

```yaml
{{.}}:
  installed: {{ if eq .Env.IMAGE_VARIANT "Minimal" }}false{{ else }}true{{ end }}
```

### Running Tests Manually

Tests can be run without Bakery. The `IMAGE_VARIANT` environment variable tells goss which packages to expect.

#### Test the Minimal Variant

```bash
# Build the minimal image first
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.min \
  -t ghcr.io/posit-dev/example-image:1.0.0-minimal \
  .

# Run dgoss with IMAGE_VARIANT=Minimal
GOSS_FILES_PATH=example-image/1.0.0/test \
dgoss run \
  -v "$(pwd)/example-image/1.0.0:/tmp/version" \
  -v "$(pwd)/example-image:/tmp/image" \
  -v "$(pwd):/tmp/project" \
  -e IMAGE_VERSION=1.0.0 \
  -e IMAGE_VARIANT=Minimal \
  -e IMAGE_VERSION_MOUNT=/tmp/version \
  -e IMAGE_MOUNT=/tmp/image \
  -e PROJECT_MOUNT=/tmp/project \
  --init \
  ghcr.io/posit-dev/example-image:1.0.0-minimal
```

#### Test the Standard Variant

```bash
# Build the standard image first
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.std \
  -t ghcr.io/posit-dev/example-image:1.0.0-standard \
  .

# Run dgoss with IMAGE_VARIANT=Standard
GOSS_FILES_PATH=example-image/1.0.0/test \
dgoss run \
  -v "$(pwd)/example-image/1.0.0:/tmp/version" \
  -v "$(pwd)/example-image:/tmp/image" \
  -v "$(pwd):/tmp/project" \
  -e IMAGE_VERSION=1.0.0 \
  -e IMAGE_VARIANT=Standard \
  -e IMAGE_VERSION_MOUNT=/tmp/version \
  -e IMAGE_MOUNT=/tmp/image \
  -e PROJECT_MOUNT=/tmp/project \
  --init \
  ghcr.io/posit-dev/example-image:1.0.0-standard
```

## Template Variables

The Containerfile template uses these Bakery variables:
- `{{ Image.Version }}` - The version being built (e.g., "1.0.0")
- `{{ Image.Variant }}` - The variant being built (e.g., "Minimal" or "Standard")
- `{{ Path.Version }}` - Path to the version directory (e.g., "example-image/1.0.0")

See [TEMPLATING.md](https://github.com/posit-dev/images-shared/blob/main/posit-bakery/TEMPLATING.md) for a list of available variables and macros.
