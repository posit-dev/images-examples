# Matrix images example

This example demonstrates how to use Bakery's matrix feature to build multiple image variants from a single set of templates. Instead of creating separate version directories for each configuration, you define a matrix of dependency combinations in `bakery.yaml` and Bakery generates images for all combinations using Docker build arguments.

All command examples are expected to run with this example, `bakery/06-matrix-images/`, as the working directory.

Bakery commands can also use the `--context PATH` option to specify the path to the example directory when running from a different location.

## Structure

```
06-matrix-images/
├── bakery.yaml                              # Project config with matrix definition
└── test-matrix/
    ├── template/                            # Jinja2 templates using build args
    │   ├── Containerfile.ubuntu2404.jinja2
    │   └── test/goss.yaml.jinja2
    └── matrix/                              # Generated files (single set for all combos)
        ├── Containerfile.ubuntu2404
        └── test/goss.yaml
```

## What this example builds

- **Base**: Ubuntu 24.04
- **R versions**: 2 most recent (resolved at version creation time)
- **Python versions**: 2 most recent (resolved at version creation time)
- **Quarto**: Latest version (resolved at version creation time)
- **Produces**: 4 images from the combination matrix (2 R × 2 Python × 1 Quarto)

## Concepts

This example demonstrates Bakery's matrix feature, which creates multiple image variants from dependency combinations without separate version directories.

### Matrix configuration in bakery.yaml

The [`matrix`][ImageMatrix] block replaces the `versions` block when you want to build multiple image variants:

```yaml
images:
  - name: test-matrix
    matrix:
      dependencyConstraints:
        - dependency: R
          constraint:
            count: 2
            latest: true
        - dependency: python
          constraint:
            count: 2
            latest: true
        - dependency: quarto
          constraint:
            latest: true
      os:
        - name: "Ubuntu 24.04"
          primary: true
```

### Three ways to define matrix values

#### 1. Dependency constraints (automatic resolution)

Use [`dependencyConstraints`][DependencyConstraint] to automatically resolve versions at creation time. Below are some examples of different constraint configurations:

Include only the latest version:
```yaml
matrix:
  dependencyConstraints:
    - dependency: R
      constraint:
        latest: true      # Include the latest version
```

Include the 2 most recent minor versions:
```yaml
matrix:
  dependencyConstraints:
    - dependency: R
      constraint:
        latest: true     # Ensure the latest version is included
        count: 2         # Include the 2 most recent versions
```

Include all minor versions at their most recent patch through the latest release starting from 4.1.0 onwards:
```yaml
matrix:
  dependencyConstraints:
    - dependency: R
      constraint:
        latest: true     # Ensure the latest version is included
        min: "4.1.0"     # Minimum version to start at
```

Include 3 minor versions up to and including 4.4.3:
```yaml
matrix:
  dependencyConstraints:
    - dependency: R
      constraint:
        max: "4.4.3"     # Maximum version to end at
        count: 3         # Include 3 versions counting backwards from the max
```

#### 2. Explicit dependencies

Use [`dependencies`][DependencyVersions] to pin specific versions:

```yaml
matrix:
  dependencies:
    - dependency: R
      versions: ["4.4.3", "4.3.3"]
    - dependency: python
      versions: ["3.12.0", "3.11.0"]
```

#### 3. Custom values

Use `values` for arbitrary key-value pairs (non-dependency variables):

```yaml
matrix:
  values:
    - CUSTOM_VAR: ["option1", "option2"]
```

### Template differences from versioned images

Matrix templates use build argument macros instead of direct version references to reuse a single definition:

```jinja2
{%- import "python.j2" as python -%}
{%- import "r.j2" as r -%}
{%- import "quarto.j2" as quarto -%}

# Build Python using uv - note the build_arg() calls
{{ python.build_stage(python.build_arg(), prerun=python.declare_build_arg()) }}

FROM docker.io/library/ubuntu:24.04

### ARG declarations ###
{{ r.declare_build_arg() }}
{{ python.declare_build_arg() }}
{{ quarto.declare_build_arg() }}

# Install R using build arg
{{ r.run_install(r.build_arg()) }}

# Install Quarto using build arg
{{ quarto.run_install(quarto.build_arg(), True) }}
```

The key macros for matrix builds:
- `python.build_arg()` - Returns `$PYTHON_VERSION` for use in Containerfile commands
- `python.declare_build_arg()` - Generates `ARG PYTHON_VERSION`
- Same pattern for `r.*` and `quarto.*`

### Generated directory structure

Matrix images use a `matrix/` subdirectory instead of version directories:

| Feature | Versioned Images | Matrix Images |
|---------|------------------|---------------|
| Output directory | `{image}/{version}/` | `{image}/matrix/` |
| Containerfiles | One per version | Single shared Containerfile |
| Version values | Hardcoded in generated files | Passed as build args at build time |

### Image tagging

Matrix images are tagged with all dependency versions. For example:
- `test-matrix-quarto1-8-27-r4-5-2-python3-14-3-ubuntu-24-04`

## Creation of this example

```bash
# Initialize a new Bakery project
bakery create project

# Create the image
bakery create image test-matrix

# Edit bakery.yaml to add matrix configuration instead of versions
# Edit templates to use build arg macros instead of Dependencies.*

# Render templates to the matrix/ directory
bakery update files
```

## Building with Bakery CLI

Bakery manages the full lifecycle of rendering templates, building images, and running tests.

```bash
# Rerender templates to generate files
bakery update files

# Build all matrix combinations
bakery build

# Run tests for all matrix combinations
bakery run dgoss
```

## Building directly with Docker

You can build matrix images directly using Docker by passing version values as build arguments:

```bash
docker buildx build \
  --load \
  --build-arg R_VERSION=4.5.2 \
  --build-arg PYTHON_VERSION=3.14.3 \
  --build-arg QUARTO_VERSION=1.8.27 \
  -f test-matrix/matrix/Containerfile.ubuntu2404 \
  -t ghcr.io/posit-dev/test-matrix:r4.5.2-python3.14.3-quarto1.8.27-ubuntu-24.04 \
  .
```

Note that you must provide the version values that Bakery would normally resolve. Check `bakery.yaml` or run `bakery build --dry-run` to see the resolved versions.

## Testing with dgoss

[Goss](https://github.com/goss-org/goss) is a serverspec-like tool for validating server configuration. [dgoss](https://github.com/goss-org/goss/tree/master/extras/dgoss) is a wrapper for testing Docker containers.

Matrix tests access version information through `BUILD_ARG_*` environment variables that Bakery sets automatically:

```yaml
file:
  python-{{ .Env.BUILD_ARG_PYTHON_VERSION }}:
    exists: true
    path: /opt/python/{{ .Env.BUILD_ARG_PYTHON_VERSION }}
  r-{{ .Env.BUILD_ARG_R_VERSION }}:
    exists: true
    path: /opt/R/{{ .Env.BUILD_ARG_R_VERSION }}
  quarto-{{ .Env.BUILD_ARG_QUARTO_VERSION }}:
    exists: true
    path: /opt/quarto/{{ .Env.BUILD_ARG_QUARTO_VERSION }}
```

The `goss.j2` macro `goss.build_arg_env_var("R_VERSION")` generates `{{ .Env.BUILD_ARG_R_VERSION }}` for use in test templates.

## Key differences from previous examples

| Feature | Examples 01-05 | Example 06 (Matrix) |
|---------|----------------|---------------------|
| Version definition | `versions:` list | `matrix:` block |
| Generated directory | `{image}/{version}/` | `{image}/matrix/` |
| Version values | Hardcoded in templates via `Dependencies.*` | Docker build args via `*.build_arg()` |
| Multiple configurations | Separate Containerfile per version | Single Containerfile with build args |
| Use case | Sequential releases | Combinatorial variants |

## Template variables

The templates use these Bakery macros for matrix builds:

**Build Arg Declaration:**

| Macro | Output |
|:------|:-------|
| `{{ r.declare_build_arg() }}` | `ARG R_VERSION` |
| `{{ python.declare_build_arg() }}` | `ARG PYTHON_VERSION` |
| `{{ quarto.declare_build_arg() }}` | `ARG QUARTO_VERSION` |

**Build Arg Usage:**

| Macro | Output |
|:------|:-------|
| `{{ r.build_arg() }}` | `$R_VERSION` |
| `{{ python.build_arg() }}` | `$PYTHON_VERSION` |
| `{{ quarto.build_arg() }}` | `$QUARTO_VERSION` |

**Test Templates:**

| Macro | Output |
|:------|:-------|
| `{{ goss.build_arg_env_var("R_VERSION") }}` | `{{ .Env.BUILD_ARG_R_VERSION }}` |

See [TEMPLATING.md](https://github.com/posit-dev/images-shared/blob/main/posit-bakery/TEMPLATING.md) for a complete list of available variables and macros.

[ImageMatrix]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#imagematrix
[DependencyConstraint]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#dependencyconstraint
[DependencyVersions]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#dependencyversions
