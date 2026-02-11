# Images with Managed Dependencies Example

This example demonstrates how to use Bakery's managed dependency system for automatic version management of R, Python, and Quarto. Instead of hardcoding dependency versions in templates, you define constraints in `bakery.yaml` and Bakery resolves them to specific versions. This allows Bakery to automatically update versions when you create new versions of your image, without needing to edit templates, and pins existing versions to what was latest at creation time to avoid shifting the image's contents after release.

All command examples are expected to run with this example, `bakery/05-images-with-managed-dependencies/`, as the working directory.

## Structure

```
05-images-with-managed-dependencies/
├── bakery.yaml                    # Project configuration with dependency constraints
└── example-image/
    ├── template/                  # Jinja2 templates using dependency macros
    │   ├── Containerfile.jinja2
    │   └── test/goss.yaml.jinja2
    └── 1.0.0/                     # Generated files for version 1.0.0
        ├── Containerfile
        └── test/goss.yaml
```

## What This Example Builds

- **Base**: Ubuntu 24.04
- **R**: 4.5.2 (latest at time of creation)
- **Python**: 3.14.3, 3.13.12 (latest 2 minor versions at time of creation)
- **Version**: 1.0.0
- **Produces**: `example-image:1.0.0`

## Concepts

This example demonstrates Bakery's managed dependency system, which separates dependency rules from specific versions.

### Dependency Constraints in bakery.yaml

[Dependency constraints][DependencyConstraint] define rules for which versions to include, rather than specifying exact versions:

```yaml
images:
  - name: example-image
    dependencyConstraints:
      - dependency: R
        constraint:
          latest: true
      - dependency: python
        constraint:
          latest: true
          count: 2
```

Constraints support:
- `latest: true` - Include the latest version
- `count: N` - Include the N most recent versions (defaults to 1)
- `min: version` - Minimum version
- `max: version` - Maximum version

Bakery currently supports managed dependencies for R, Python, and Quarto.

### Automatic Version Resolution

When you run `bakery create version`, Bakery resolves constraints to specific versions. This means the resolved versions reflect what was latest at the time the version was created, not at build time.

For example, the constraint `latest: true` with `count: 2` for Python resolved to:
- Python 3.14.3 (latest)
- Python 3.13.12 (second latest)

### Dependencies Field (Generated)

Bakery automatically populates the [`dependencies`][DependencyVersions] field in each version from constraints:

```yaml
versions:
  - name: 1.0.0
    latest: true
    dependencies:
      - dependency: R
        version: "4.5.2"
      - dependency: python
        versions: ["3.14.3", "3.13.12"]
```

You can also manually specify this field to pin specific versions rather than using automatic resolution.

### Template Variables for Dependencies

Templates access resolved dependencies through these variables:

| Variable | Description |
|:---------|:------------|
| `Dependencies.R` | List of R versions |
| `Dependencies.python` | List of Python versions |
| `Dependencies.quarto` | List of Quarto versions |

These are lists, allowing iteration for multi-version installs:

```jinja2
{% for r_version in Dependencies.R -%}
# Install R {{ r_version }}
{% endfor %}
```

### Bakery Macros for Installation

Bakery provides macros that handle dependency installation with best practices:

**Python (python.j2)**:

| Macro | Description |
|:------|:------------|
| `python.build_stage(versions)` | Creates a multi-stage build using uv to install Python |
| `python.copy_from_build_stage()` | Copies installed Python from the builder stage |
| `python.get_version_directory(version)` | Returns the path to a Python version |

**R (r.j2)**:

| Macro | Description |
|:------|:------------|
| `r.run_install(versions)` | Installs R versions using the official installer |
| `r.get_version_directory(version)` | Returns the path to an R version |

**Quarto (quarto.j2)** (when needed):

| Macro | Description |
|:------|:------------|
| `quarto.run_install(versions)` | Installs Quarto versions |

The Containerfile template uses these macros:

```jinja2
{%- import "python.j2" as python -%}
{%- import "r.j2" as r -%}

# Build Python using uv in a separate stage
{{ python.build_stage(Dependencies.python) }}

FROM docker.io/library/ubuntu:24.04
...
{{ python.copy_from_build_stage() }}
{{ r.run_install(Dependencies.R) }}
```

## Creation of this Example

```bash
# Initialize a new Bakery project
bakery create project

# Create the image
bakery create image example-image

# Edit bakery.yaml to add dependencyConstraints
# Edit templates to use dependency macros

# Create a version - this resolves constraints to specific versions
bakery create version example-image 1.0.0
```

The `bakery create version` command queries the latest available versions and populates the `dependencies` field automatically.

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

## Building Directly with Docker

You can build directly using Docker. Note the multi-stage build for Python installation.

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

The test template iterates over dependencies to verify each version is installed:

```yaml
command:
  {% for r_version in Dependencies.R -%}
  "Verify R installation is version {{ r_version }}":
    exec: {{ r.get_version_directory(r_version) }}/bin/R --version
    exit-status: 0
    stdout:
      - "R version {{ r_version }}"
  {%- endfor %}
```

### Running Tests Manually

```bash
# Build the image first
docker buildx build \
  --load \
  -f example-image/1.0.0/Containerfile \
  -t ghcr.io/posit-dev/example-image:1.0.0 \
  .

# Run dgoss
GOSS_FILES_PATH=example-image/1.0.0/test \
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

## Key Differences from Previous Examples

| Feature | Examples 01-04 | Example 05 |
|---------|----------------|------------|
| Dependency declaration | Manual in templates | `dependencyConstraints` in bakery.yaml |
| Package files | `deps/packages.txt` | Not needed for managed deps |
| Installation commands | Manual apt/pip/R commands | Bakery macros (python.j2, r.j2) |
| Version management | Hardcoded in templates | Auto-resolved from constraints |

## Template Variables

The templates use these Bakery variables:

| Variable | Description | Example |
|:---------|:------------|:--------|
| `{{ Image.Version }}` | The version being built | `"1.0.0"` |
| `{{ Path.Version }}` | Path to the version directory | `"example-image/1.0.0"` |
| `{{ Dependencies.R }}` | List of R versions to install | `["4.5.2"]` |
| `{{ Dependencies.python }}` | List of Python versions to install | `["3.14.3", "3.13.12"]` |
| `{{ Dependencies.quarto }}` | List of Quarto versions to install (when used) | `["1.8.27"]` |

See [TEMPLATING.md](https://github.com/posit-dev/images-shared/blob/main/posit-bakery/TEMPLATING.md) for a complete list of available variables and macros.

[DependencyConstraint]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#dependencyconstraint
[DependencyVersions]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#dependencyversions
