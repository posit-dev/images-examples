# Extending a fleet of Posit product images

This example shows how a team can manage a small fleet of Posit product images on top of the official [Minimal](https://github.com/posit-dev/images/blob/main/docs/products/standard-vs-minimal.md) (`-min`) bases. You can version, customize, and rebuild three images (Posit Workbench, Posit Connect, and Posit Package Manager) as a single project. This keeps your development environment, deployment runtime, and package server in lockstep.

The sibling [`extending/`](..) examples each show one customization of a single Posit image in a standalone Containerfile. This example covers the same kind of customization at fleet scale (multiple Posit products in one project), using [Posit Bakery](https://posit-dev.github.io/images-shared/) to manage rendering, versioning, and tagging across the fleet. The Bakery [tutorial examples](../../bakery/) cover its features in isolation. This one applies them to a realistic team setup.

Run all command examples with `extending/posit-team/` as the working directory.

Bakery commands can also use the `--context PATH` option to specify the path to the example directory when running from a different location.

### Bakery documentation

- [Bakery guide](https://posit-dev.github.io/images-shared/): project workflow and CLI concepts
- [Configuration reference](https://posit-dev.github.io/images-shared/configuration.html): `bakery.yaml`, images, versions, OSes, and dependency constraints
- [Templating and macros](https://posit-dev.github.io/images-shared/templating.html): template variables and package-installation macros

## Structure

```text
posit-team/
├── bakery.yaml                                  # Project config: 3 images, shared R and Python constraints
├── workbench/
│   ├── template/                                # Source templates
│   │   ├── Containerfile.ubuntu2404.jinja2
│   │   ├── deps/ubuntu-24.04_packages.txt.jinja2
│   │   ├── deps/{r,python}-packages.txt.jinja2
│   │   └── test/goss.yaml.jinja2
│   └── 2026.09/                                 # Generated files
│       ├── Containerfile.ubuntu2404
│       ├── deps/ubuntu-24.04_packages.txt
│       ├── deps/{r,python}-packages.txt
│       └── test/goss.yaml
├── connect/
│   ├── template/
│   └── 2026.09/
└── package-manager/
    ├── template/
    │   ├── Containerfile.ubuntu2404.jinja2
    │   ├── ca/Example-RootCA.crt                # Static asset (copied as-is)
    │   └── test/goss.yaml.jinja2
    └── 2026.09/
```

## What this example builds

| Image | Base | Adds |
|:------|:-----|:-----|
| `workbench:2026.09.0-174.pro3` | `posit/workbench:2026.09.0-174.pro3-ubuntu-24.04-min` | R 4.6.1, Python 3.14.7, team R + Python packages, spatial system deps |
| `connect:2026.09.0` | `posit/connect:2026.09.0-ubuntu-24.04-min` | Same R, Python, and packages as Workbench |
| `package-manager:2026.09.0` | `posit/package-manager:2026.09.0-ubuntu-24.04-min` | Internal certificate authority (CA) certificate in the system trust store |

Package Manager does not host user code, so it gets a much lighter customization than Workbench and Connect.

> Why install R 4.6.1 and Python 3.14.7 when the 2026.09 Standard images already contain them? This example deliberately uses Minimal bases to show how a team can own the language layer. The team can keep the versions aligned with the current Posit release, as here, or roll R and Python forward or hold them back independently of Posit.

## Concepts

### Fleet versioning maps to Posit product versions

Each image's [`Image.Version`](https://posit-dev.github.io/images-shared/templating.html) is the Posit product version it extends. The `versions` and `subpath` fields are defined by the [image-version configuration](https://posit-dev.github.io/images-shared/configuration.html#imageversion) in Bakery:

```jinja2
FROM docker.io/posit/workbench:{{ Image.Version | tagSafe }}-ubuntu-24.04-min
```

When Posit releases a new product version, the team appends a new entry under `versions:` and runs `bakery update files`. The Posit product version is not a Bakery-managed dependency, so the team owns when to roll it forward (typically tracked in their own change process).

Different products release on different cadences and Workbench includes build metadata in its version. In this example, Workbench uses `2026.09.0+174.pro3`, while Connect and Package Manager use `2026.09.0`. Each image's `versions:` list is independent, and `subpath: "2026.09"` keeps the generated files organized by the product release line.

### Shared R and Python pinning across Workbench and Connect

Workbench and Connect share R and Python pins so that anything a developer builds in Workbench will run in Connect without dependency surprises. The constraint declaration is identical between the two images:

```yaml
images:
  - name: workbench
    dependencyConstraints:
      - dependency: R
        constraint:
          latest: true
      - dependency: python
        constraint:
          latest: true
    versions:
      - name: 2026.09.0+174.pro3
        subpath: "2026.09"
        dependencies:
          - dependency: R
            version: "4.6.1"
          - dependency: python
            version: "3.14.7"
  - name: connect
    dependencyConstraints: # same as workbench
      - dependency: R
        constraint:
          latest: true
      - dependency: python
        constraint:
          latest: true
    versions:
      - name: 2026.09.0
        subpath: "2026.09"
        dependencies: # same as workbench
          - dependency: R
            version: "4.6.1"
          - dependency: python
            version: "3.14.7"
```

`bakery create version` resolves an image's [dependency constraints](https://posit-dev.github.io/images-shared/configuration.html#dependencyconstraint) once, then writes the resolved values into that version's [`dependencies`](https://posit-dev.github.io/images-shared/configuration.html#dependencyversions) block. From that point on, the version is pinned, and re-running the command on a different day will not change the existing entry.

Bakery does not enforce sync across images. `dependencyConstraints` is per-image, and two images with identical `latest: true` constraints will diverge if you create your versions on different days. Keeping `workbench` and `connect` aligned is part of the team's workflow, not something Bakery guarantees:

- Create both versions in the same command sequence so the resolved R and Python land on the same values, or
- Resolve once for `workbench`, then copy the resolved `dependencies:` block into the new version of `connect` by hand.

If the team adds more images later (e.g., a content runtime), the same constraint block is the starting point. The same manual-sync discipline applies.

### Per-image customization where it matters

Each image's template carries the customizations specific to that product:

- `workbench`: installs the team system-package delta, R 4.6.1, Python 3.14.7, and team R and Python packages.
- `connect`: installs the same system packages and language packages as Workbench so apps developed in Workbench deploy cleanly.
- `package-manager`: adds a single CA certificate to the trust store. It does not install R or Python because Package Manager does not run user code.

The fleet system-package file is a small addition to the Minimal base, not a copy of the Standard image package inventory. It covers the native libraries needed by the example's spatial and graphics packages:

| Category | Packages |
|:---------|:---------|
| Spatial | `libgdal-dev`, `libgeos-dev`, `libproj-dev`, `libudunits2-dev` |
| Database and XML | `libsqlite3-dev`, `libxml2-dev`, `libcurl4-openssl-dev`, `libssl-dev` |
| Fonts and graphics | `libfontconfig1-dev`, `libfreetype-dev`, `libharfbuzz-dev`, `libfribidi-dev`, `libpng-dev`, `libtiff-dev`, `libjpeg-dev` |

The current Workbench Minimal base already supplies its compiler toolchain. Connect Minimal does not include one. Add a build toolchain to the shared list if the fleet expects packages to compile from source rather than use the Posit Public Package Manager (P3M) binaries and Python wheels used by this example.

Workbench and Connect templates are nearly identical because the team enforces that their dev and deploy environments match. The templates differ in the following ways:

1. The base image (`posit/workbench:...` vs `posit/connect:...`)
2. The goss tests (one checks for `/usr/lib/rstudio-server/bin/rserver`, the other checks for `/opt/rstudio-connect/bin/connect`)

If a team needs them to diverge (say, larger R libraries on Workbench for interactive work), the template separation makes that easy to do without affecting the other.

### Package lists are duplicated, not shared

The `ubuntu-24.04_packages.txt`, `r-packages.txt`, and `python-packages.txt` files under `workbench/template/deps/` and `connect/template/deps/` contain identical content. The system package filename makes the supported base OS explicit, matching the organization used by the product image repositories. Package Manager has no dependency list because its customization is only a certificate. Bakery has no built-in mechanism to share a deps file across images, so the team maintains the Workbench and Connect lists by hand.

In practice, the diff in `git review` catches drift: if someone edits one file and not the other, the PR shows two diffs in different image trees, or just one. Both are immediately visible. For a two-image fleet that is tolerable. For a larger fleet, consider:

- A pre-commit hook that fails if the deps files diverge.
- A `_shared/` directory with the canonical lists, then per-image template files that just `{% include %}` them (verify your Bakery version supports template paths outside the image's own `template/`).
- A separate script that regenerates per-image deps files from a single source.

The example deliberately uses the simplest form to keep the structure obvious. The duplication is the cost of that simplicity.

### Why use Bakery instead of standalone Containerfiles?

The other [extending](..) examples show the standalone Containerfile approach. That is the right starting point for a single customization. Bakery becomes valuable when:

- Multiple images need consistent R and Python versions
- Multiple Posit product versions need to coexist (e.g., maintaining `2026.09.0` and `2025.09.2` for a phased rollout)
- The team wants reproducible, version-controlled package lists per release
- Goss tests need to assert on resolved versions

This example demonstrates all four.

## Creation of this example

```bash
# Initialize a new Bakery project
bakery create project

# Create each image
bakery create image workbench
bakery create image connect
bakery create image package-manager

# Edit bakery.yaml to add dependencyConstraints and team-specific config
# Edit each image's template/Containerfile.ubuntu2404.jinja2 to FROM the Posit -min base

# Add the first version of each image, matching the Posit product version it extends
bakery create version workbench '2026.09.0+174.pro3'
bakery create version connect 2026.09.0
bakery create version package-manager 2026.09.0
```

## Building with the Bakery CLI

See the [Bakery build workflow](https://posit-dev.github.io/images-shared/#step-4-build-the-images) for the corresponding CLI lifecycle.

```bash
# Rerender templates after changes
bakery update files

# Build everything
bakery build

# Build a single image
bakery build --image workbench

# Run goss tests for every image
bakery run dgoss
```

## Building directly with Docker

Once rendered, each Containerfile is a normal Docker build context. The build context must be the example root (`extending/posit-team/`) because the Containerfile `COPY` instructions reference paths relative to it.

```bash
docker buildx build \
  --load \
  -f workbench/2026.09/Containerfile.ubuntu2404 \
  -t ghcr.io/example-org/workbench:2026.09.0-174.pro3 \
  -t ghcr.io/example-org/workbench:latest \
  .
```

## Updating to a new Posit product version

When a new Workbench version ships:

1. Add a new entry under `workbench.versions:` in `bakery.yaml`, with the Posit product version as the `name`.
2. Run `bakery create version workbench <new-version>` (or `bakery update files` if you wrote the version entry by hand).
3. Bakery resolves the current `latest` R and Python and writes them into the new version's `dependencies:` block. Existing versions stay pinned to their original R and Python values.
4. Repeat for `connect` with the same Posit product version (and matching R and Python pins).

## Adding a fourth image to the fleet

If the team wants to add, say, a content-runtime image:

1. `bakery create image content-runtime`
2. Copy `dependencyConstraints` from `workbench` to keep R and Python aligned.
3. Decide on a base, likely `posit/connect-content:<version>-min`, or extend from `connect` directly.
4. Add a version and customize the template.

The pattern scales because each image is independently described but participates in the same `bakery update files` and `bakery build` lifecycle.

## Production considerations

This example is the starting point, not the destination. Before running this in production, decide on each of the following.

### Pin the package repository to a date, not `latest`

The rendered Containerfiles install R packages from `https://p3m.dev/cran/__linux__/noble/latest`. `latest` floats: every rebuild pulls whatever P3M serves that day. For reproducible images, swap to a [P3M snapshot URL](https://docs.posit.co/rspm/admin/serving-binaries/#package-binary-urls) with a fixed date (e.g., `https://p3m.dev/cran/__linux__/noble/2026-09-15`). The team chooses when to bump the snapshot, the same way they choose when to bump the Posit product version.

The `r.run_install_packages` macro in Bakery takes the repo URL through its `_os` argument indirectly (it computes the URL from the OS codename). To pin to a snapshot, either bypass the macro and write the `install.packages` RUN command directly with the snapshot URL, or pass a custom `_os` dict whose `Codename` includes the date suffix.

The Python install path has the same issue: pip resolves from PyPI's current state at build time. Pin via a constraints file or a private mirror.

### `r.run_install_packages` with more than one R version

The example resolves to a single R version because the `dependencyConstraints` block uses bare `latest: true`. If a team adds `count: 2` to install two R minor versions, the `r.run_install_packages` macro in Bakery will emit one `RUN install.packages ... && rm -f /tmp/r-packages.txt` per version. The second `RUN` has nothing to read because the first deleted the file.

Workarounds: write the per-version install loop directly in the template (skip the macro), or open an issue against bakery to expose the `clean` parameter on `run_install_packages`. Python has the same caveat with `count: 2`, but `python.run_install_packages` does expose `clean`, so passing `clean=False` plus a manual `RUN rm` works there.

### Tag your images with the team's own version, not just the Posit product version

This example tags each image with the Posit product version it extends (`workbench:2026.09.0-174.pro3`). That is the cleanest mapping for a team that rebuilds when Posit ships and never customizes mid-cycle. If your team adds packages or system deps between Posit releases, append a team-side suffix (`workbench:2026.09.0-2`, or `workbench:2026.09.0-team-r3`). Bakery does not have an opinion here. Set `name:` under `versions:` to whatever the team's tagging scheme requires.

## Template variables

| Variable | Description | Example |
|:---------|:------------|:--------|
| `{{ Image.Version }}` | Posit product version this image extends | `"2026.09.0+174.pro3"` |
| `{{ Path.Version }}` | Path to the version directory | `"workbench/2026.09"` |
| `{{ Dependencies.R }}` | Resolved R versions | `["4.6.1"]` |
| `{{ Dependencies.python }}` | Resolved Python versions | `["3.14.7"]` |

See the [Bakery templating and macros reference](https://posit-dev.github.io/images-shared/templating.html) for the full reference.

## Related examples

- [extending/](..): the standalone Containerfile siblings of this example. Start there if you only need to customize one image.
- [bakery/01-basic-image](../../bakery/01-basic-image/): the simplest possible Bakery project, on a stock OS base. Useful for understanding the templating mechanics this example builds on.
- [bakery/05-images-with-managed-dependencies](../../bakery/05-images-with-managed-dependencies/): the `dependencyConstraints` mechanism used here to resolve R and Python.
