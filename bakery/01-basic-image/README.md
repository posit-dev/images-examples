# Basic Image Example

This example demonstrates the simplest use case for Bakery: a single [image][Image] with one [version][ImageVersion]. It builds an Ubuntu 24.04 image with basic development tools installed.

All command examples are expected to run with this example, `bakery/01-basic-image/`, as the working directory. 

Bakery commands can also use the `--context PATH` option to specify the path to the example directory when running from a different location.

## Structure

```
01-basic-image/
├── bakery.yaml                    # Project configuration
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

- **Base**: Ubuntu 24.04
- **Packages**: `build-essential`, `ca-certificates`, `curl`, `git`
- **Version**: 1.0.0

## Concepts

This example demonstrates Bakery's usage in its most basic form. It consists of a [`bakery.yaml` file][BakeryConfiguration] that defines the Bakery project configuration and a single image directory. The image directory, `example-image/`, contains a `template/` directory with Jinja2 templates and a version directory, `1.0.0/`, with generated files. 

Each time you add a new [version][ImageVersion] to the `bakery.yaml` file, Bakery renders the templates to generate the necessary files for that version. In many cases, the image's version will correlate to the primary software it packages (e.g. a product version, an R version, etc.), but in this example, the version is arbitrary and does not correspond to any software version.

## Creation of this Example

Starting a new Bakery project can be done with a couple bootstrapping commands, editing templates, and then adding the first version. The commands below were used to create this example from scratch.

```bash
# Initialize a new Bakery project in the current directory by writing a skeleton bakery.yaml file
bakery create project

# Create a new image called "example-image": adds the image to bakery.yaml, creates the image directory, and adds a skeleton set of template files for an image
bakery create image example-image

# Edit the Containerfile template and package list template to specify the desired image configuration

# Add the first version (1.0.0) to bakery.yaml: this will trigger rendering of the templates to generate files for version 1.0.0
bakery create version example-image 1.0.0
```

## Building with Bakery command-line interface (CLI)

Bakery manages the full lifecycle of rendering templates, building images, and running tests.

```bash
# Rerender templates to generate files for existing versions from templates
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

Like building, you can run tests without Bakery. The commands below would be functionally equivalent to running `bakery build` followed by `bakery run dgoss`. Always build and load the image locally, or pull it, before running tests.

Note that some options and environment variables passed to `dgoss` are included to mirror the calls made by `bakery`, but will go unused in practice.

```bash
# Build the image first to make it available for testing
docker buildx build \
  --load \
  -f bakery/01-basic-image/example-image/1.0.0/Containerfile \
  -t ghcr.io/posit-dev/example-image:1.0.0 \
  -t ghcr.io/posit-dev/example-image:latest \
  .

# Run dgoss with expected environment variables and mounts
GOSS_FILES_PATH=bakery/01-basic-image/example-image/1.0.0/test \
dgoss run \
  -v "$(pwd)/example-image/1.0.0:/tmp/version" \
  -v "$(pwd)/example-image:/tmp/image" \
  -v "$(pwd):/tmp/project" \
  -e IMAGE_VERSION=1.0.0 \
  -e IMAGE_VERSION_MOUNT=/tmp/version \
  -e IMAGE_MOUNT=/tmp/image \
  -e PROJECT_MOUNT=/tmp/project \
  --init \
  ghcr.io/posit-dev/example-image:1.0.0
```

## Template Variables

The Containerfile template uses these Bakery variables:

| Variable | Description | Example |
|:---------|:------------|:--------|
| `{{ Image.Version }}` | The version being built | `"1.0.0"` |
| `{{ Path.Version }}` | Path to the version directory | `"example-image/1.0.0"` |

See [TEMPLATING.md](https://github.com/posit-dev/images-shared/blob/main/posit-bakery/TEMPLATING.md) for a list of available variables and macros.

[BakeryConfiguration]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#bakery-configuration
[Image]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#image
[ImageVersion]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#imageversion
