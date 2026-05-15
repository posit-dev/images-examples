# Image variants example

This example demonstrates how to build multiple variants of the same image from a single template. It creates both minimal and standard variants of an Ubuntu 24.04 image, each with different package sets.

Run all command examples from `bakery/03-image-variants/` as the working directory.

Bakery commands can also use the `--context PATH` option to specify the path to the example directory when running from a different location.

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

## What this example builds

- **Base**: Ubuntu 24.04
- **Variants**:
  - **Minimal**: Core packages only (`build-essential`, `ca-certificates`, `curl`, `git`)
  - **Standard**: Core + optional packages (`libodbc2`, `libpq-dev`)
- **Version**: 1.0.0
- **Produces**: `example-image:1.0.0-min` and `example-image:1.0.0-std`

## Concepts

### Variant configuration in bakery.yaml

Define variants in the image configuration with the `name`, `extension`, `tagDisplayName`, and `primary` fields:

```yaml
images:
  - name: example-image
    variants:
      - name: Standard
        extension: std
        tagDisplayName: std
        primary: true
      - name: Minimal
        extension: min
        tagDisplayName: min
    versions:
      - name: 1.0.0
        latest: true
```

- **name**: Human-readable variant name, accessible as `{{ Image.Variant }}` in templates
- **extension**: Suffix appended to generated filenames (e.g., `Containerfile.std`)
- **tagDisplayName**: The variant string used in image tags (e.g., `example-image:1.0.0-std`). Defaults to the lowercased `name` if omitted.
- **primary**: Marks the default variant; Bakery also adds shorter "primary variant" tags (e.g., `example-image:1.0.0`) that omit the variant suffix

### Conditional template logic

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

This pattern allows a single template to generate different Containerfiles based on which variant Bakery renders.

### Variant-specific output files

Bakery generates separate Containerfiles for each variant, distinguished by their extensions:

- `Containerfile.min` - Standalone build file for the Minimal variant
- `Containerfile.std` - Standalone build file for the Standard variant

Each generated file is complete, and you can build them independently.

### Package separation pattern

This example separates packages into two files:
- `packages.txt` - Core packages installed in all variants
- `optional_packages.txt` - Additional packages installed only in the Standard variant

This pattern keeps package lists maintainable and makes variant differences explicit.

## Creation of this example

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

## Building directly with Docker

You can build each variant directly using Docker. The build context must be the example directory because the Containerfile references paths relative to it.

### Build the minimal variant

```bash
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.min \
  -t ghcr.io/posit-dev/example-image:1.0.0-min \
  -t ghcr.io/posit-dev/example-image:min \
  .
```

### Build the standard variant

```bash
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.std \
  -t ghcr.io/posit-dev/example-image:1.0.0-std \
  -t ghcr.io/posit-dev/example-image:std \
  -t ghcr.io/posit-dev/example-image:latest \
  .
```

## Testing with dgoss

[Goss](https://github.com/goss-org/goss) is a serverspec-like tool for validating server configuration. [dgoss](https://github.com/goss-org/goss/tree/master/extras/dgoss) is a wrapper for testing Docker containers.

The goss.yaml template uses the `IMAGE_VARIANT` environment variable to check whether optional packages are present:

```yaml
{{.}}:
  installed: {{ if eq .Env.IMAGE_VARIANT "Minimal" }}false{{ else }}true{{ end }}
```

### Run tests without Bakery

You can run tests without Bakery. The `IMAGE_VARIANT` environment variable tells goss which packages to expect.

#### Test the minimal variant

```bash
# Build the minimal image first
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.min \
  -t ghcr.io/posit-dev/example-image:1.0.0-min \
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
  ghcr.io/posit-dev/example-image:1.0.0-min
```

#### Test the standard variant

```bash
# Build the standard image first
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile.std \
  -t ghcr.io/posit-dev/example-image:1.0.0-std \
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
  ghcr.io/posit-dev/example-image:1.0.0-std
```

## Template variables

The Containerfile template uses these Bakery variables:

| Variable | Description | Example |
|:---------|:------------|:--------|
| `{{ Image.Version }}` | The version being built | `"1.0.0"` |
| `{{ Image.Variant }}` | The variant being built | `"Minimal"`, `"Standard"` |
| `{{ Path.Version }}` | Path to the version directory | `"example-image/1.0.0"` |

See the [Bakery templating documentation](https://posit-dev.github.io/images-shared/templating.html) for a list of available variables and macros.

[ImageVariant]: https://posit-dev.github.io/images-shared/configuration.html#imagevariant
