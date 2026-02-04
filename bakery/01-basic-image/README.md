# Basic Image Example

This example demonstrates the simplest use case for Bakery: a single image with one version. It builds an Ubuntu 22.04 image with basic development tools installed.

All command examples are expected to run with this example, `bakery/01-basic-image/`, as the working directory.

## Structure

```
01-basic-image/
├── bakery.yaml                    # Repository configuration
└── example-image/
    ├── template/                  # Jinja2 templates (source of truth)
    │   ├── Containerfile.jinja2
    │   ├── deps/packages.txt.jinja2
    │   └── test/goss.yaml.jinja2
    └── 1.0.0/                     # Generated files for version 1.0.0
        ├── Containerfile
        ├── deps/packages.txt
        └── test/goss.yaml
```

## What This Example Builds

- **Base**: Ubuntu 22.04
- **Packages**: `build-essential`, `ca-certificates`, `curl`, `git`
- **Version**: 1.0.0

## Building with Bakery CLI

Bakery manages the full lifecycle of rendering templates, building images, and running tests.

```bash
# Render templates to generate version-specific files
bakery update files

# Build the image
bakery build

# Run tests
bakery run dgoss
```

## Building Directly with Docker

You can build the image directly using Docker without Bakery. The build context must be the example directory (not the version directory) because the Containerfile references paths relative to it.

Running one of the below commands will be functionally equivalent to running `bakery build`.

```bash
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile \
  -t ghcr.io/posit-dev/example-image:1.0.0 \
  -t ghcr.io/posit-dev/example-image:latest \
  .
```

## Testing with dgoss

[Goss](https://github.com/goss-org/goss) is a serverspec-like tool for validating server configuration. [dgoss](https://github.com/goss-org/goss/tree/master/extras/dgoss) is a wrapper for testing Docker containers.

The goss.yaml file uses Go templates to dynamically read the package list and verify each package is installed.

### Running Tests Manually

Like building, tests can be run without the assistance of Bakery. The commands below would be functionally equivalent to running `bakery build` followed by `bakery run dgoss`. The image should always be built and loaded locally or pulled prior to running tests.

Note that some options and environment variables passed to 

```bash
# Build the image first to make it available for testing
docker buildx build \
  --load \
  -f bakery/01-basic-image/example-image/1.0.0/Containerfile \
  -t ghcr.io/posit-dev/example-image:1.0.0 \
  -t ghcr.io/posit-dev/example-image:latest \
  bakery/01-basic-image/

# Run dgoss with expected environment variables and mounts
GOSS_FILES_PATH=bakery/01-basic-image/example-image/1.0.0/test \
dgoss run \
  -v "$(pwd)/bakery/01-basic-image/example-image/1.0.0:/tmp/version" \
  -v "$(pwd)/bakery/01-basic-image/example-image:/tmp/image" \
  -v "$(pwd)/bakery/01-basic-image:/tmp/project" \
  -e IMAGE_VERSION=1.0.0 \
  -e IMAGE_VERSION_MOUNT=/tmp/version \
  -e IMAGE_MOUNT=/tmp/image \
  -e PROJECT_MOUNT=/tmp/project \
  --init \
  ghcr.io/posit-dev/example-image:1.0.0
```

## Template Variables

The Containerfile template uses these Bakery variables:
- `{{ Image.Version }}` - The version being built (e.g., "1.0.0")
- `{{ Path.Version }}` - Path to the version directory (e.g., "example-image/1.0.0")

See [TEMPLATING.md](https://github.com/posit-dev/images-shared/blob/main/posit-bakery/TEMPLATING.md) for a list of available variables and macros.