# Extending a fleet of Posit product images

This example shows how a team can manage a small fleet of Posit product images on top of the official [Minimal](https://github.com/posit-dev/images/blob/main/docs/products/standard-vs-minimal.md) (`-min`) bases. Three images (Posit Workbench, Posit Connect, and Posit Package Manager) are versioned, customized, and rebuilt as a single project so the team's development environment, deployment runtime, and package server stay in lockstep.

The sibling [`extending/`](..) examples each show one customization of a single Posit image in a standalone Containerfile. This example covers the same kind of customization at fleet scale (multiple Posit products in one project), using the [Bakery](../../bakery/) tool from Posit to manage rendering, versioning, and tagging across the fleet. The Bakery [tutorial examples](../../bakery/) cover its features in isolation. This one applies them to a realistic team setup.

Run all command examples with `extending/posit-team/` as the working directory.

Bakery commands can also use the `--context PATH` option to specify the path to the example directory when running from a different location.

## Structure

```
posit-team/
├── bakery.yaml                                  # Project config: 3 images, shared R and Python constraints
├── workbench/
│   ├── template/                                # Source templates
│   │   ├── Containerfile.jinja2
│   │   ├── deps/{system,r,python}-packages.txt.jinja2
│   │   └── test/goss.yaml.jinja2
│   └── 2026.04.0/                               # Generated files
│       ├── Containerfile
│       ├── deps/{system,r,python}-packages.txt
│       └── test/goss.yaml
├── connect/
│   ├── template/
│   └── 2026.04.0/
└── package-manager/
    ├── template/
    │   ├── Containerfile.jinja2
    │   ├── ca/Example-RootCA.crt                # Static asset (copied as-is)
    │   └── test/goss.yaml.jinja2
    └── 2026.04.1/
```

## What this example builds

| Image | Base | Adds |
|:------|:-----|:-----|
| `workbench:2026.04.0` | `posit/workbench:2026.04.0-ubuntu-24.04-min` | R 4.5.3, Python 3.14.4, team R + Python packages, spatial system deps |
| `connect:2026.04.0` | `posit/connect:2026.04.0-ubuntu-24.04-min` | Same R, Python, and packages as Workbench |
| `package-manager:2026.04.1` | `posit/package-manager:2026.04.1-ubuntu-24.04-min` | Internal CA certificate in the system trust store |

Package Manager does not host user code, so it gets a much lighter customization than Workbench and Connect.

> **Why R 4.5.3 when Connect 2026.04.0 Standard ships R 4.6.0?** This is the whole reason for choosing the Minimal base over Standard. The Posit Standard variants ship whichever R and Python were current when that product version was cut, and those pins move with each Posit release. A team that wants to decouple its R and Python release cadence from the Posit product release cadence installs the languages itself on top of `-min`. Here, `workbench` and `connect` are pinned to R 4.5.3 and Python 3.14.4 (the versions Workbench 2026.04.0 Standard shipped). The team can roll R forward or hold it back independently of Posit.

## Concepts

### Fleet versioning maps to Posit product versions

Each image's `Image.Version` is the Posit product version it extends:

```jinja2
FROM docker.io/posit/workbench:{{ Image.Version }}-ubuntu-24.04-min
```

When Posit releases a new product version, the team appends a new entry under `versions:` and runs `bakery update files`. The Posit product version is not a Bakery-managed dependency, so the team owns when to roll it forward (typically tracked in their own change process).

Different products release on different cadences. In this example, Workbench and Connect both use `2026.04.0`, while Package Manager rolled to `2026.04.1`. Each image's `versions:` list is independent.

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
      - name: 2026.04.0
        dependencies:
          - dependency: R
            version: "4.5.3"
          - dependency: python
            version: "3.14.4"
  - name: connect
    dependencyConstraints: # same as workbench
      - dependency: R
        constraint:
          latest: true
      - dependency: python
        constraint:
          latest: true
    versions:
      - name: 2026.04.0
        dependencies: # same as workbench
          - dependency: R
            version: "4.5.3"
          - dependency: python
            version: "3.14.4"
```

`bakery create version` resolves an image's constraints once, then writes the resolved values into that version's `dependencies:` block. From that point on, the version is pinned, and re-running the command on a different day will not change the existing entry.

Bakery does not enforce sync across images. `dependencyConstraints` is per-image, and two images with identical `latest: true` constraints will diverge if their versions are created on different days. Keeping `workbench` and `connect` aligned is part of the team's workflow, not something Bakery guarantees:

- Create both versions in the same command sequence so the resolved R and Python land on the same values, or
- Resolve once for `workbench`, then copy the resolved `dependencies:` block into the new version of `connect` by hand.

If the team adds more images later (e.g., a content runtime), the same constraint block is the starting point. The same manual-sync discipline applies.

### Per-image customization where it matters

Each image's template carries the customizations specific to that product:

- `workbench`: system dependencies for spatial packages, R, Python, team R packages, and team Python packages.
- `connect`: the same R, Python, and packages as Workbench, plus a comment in the template noting the alignment requirement.
- `package-manager`: a single CA certificate copied into the system trust store. No R, no Python. Package Manager does not run user code.

Workbench and Connect templates are nearly identical because the team enforces that their dev and deploy environments match. The differences are in:

1. The base image (`posit/workbench:...` vs `posit/connect:...`)
2. The goss tests (one checks for `/usr/lib/rstudio-server/bin/rserver`, the other checks for `/opt/rstudio-connect/bin/connect`)

If a team needs them to diverge (say, larger R libraries on Workbench for interactive work), the template separation makes that easy to do without affecting the other.

### Package lists are duplicated, not shared

The `system-packages.txt`, `r-packages.txt`, and `python-packages.txt` files under `workbench/template/deps/` and `connect/template/deps/` contain identical content. Bakery has no built-in mechanism to share a deps file across images, so the team maintains them by hand.

In practice the diff in `git review` catches drift: if someone edits one file and not the other, the PR shows two diffs in different image trees, or just one. Both are immediately visible. For a two-image fleet that is tolerable. For a larger fleet, consider:

- A pre-commit hook that fails if the deps files diverge.
- A `_shared/` directory with the canonical lists, then per-image template files that just `{% include %}` them (verify your Bakery version supports template paths outside the image's own `template/`).
- A separate script that regenerates per-image deps files from a single source.

The example deliberately uses the simplest form to keep the structure obvious. The duplication is the cost of that simplicity.

### Why use Bakery instead of standalone Containerfiles?

The other [extending](..) examples show the standalone Containerfile approach. That is the right starting point for a single customization. Bakery becomes valuable when:

- Multiple images need consistent R and Python versions
- Multiple Posit product versions need to coexist (e.g., maintaining `2026.04.0` and `2025.09.2` for a phased rollout)
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
# Edit each image's template/Containerfile.jinja2 to FROM the Posit -min base

# Add the first version of each image, matching the Posit product version it extends
bakery create version workbench 2026.04.0
bakery create version connect 2026.04.0
bakery create version package-manager 2026.04.1
```

## Building with the Bakery CLI

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
  -f workbench/2026.04.0/Containerfile \
  -t ghcr.io/example-org/workbench:2026.04.0 \
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

The pattern scales because each image is independently described but participates in the same `bakery update files` / `bakery build` lifecycle.

## Production considerations

This example is the starting point, not the destination. Before running this in production, decide on each of the following.

### Pin the package repository to a date, not `latest`

The rendered Containerfiles install R packages from `https://p3m.dev/cran/__linux__/noble/latest`. `latest` floats: every rebuild pulls whatever P3M serves that day. For reproducible images, swap to a [P3M snapshot URL](https://docs.posit.co/rspm/admin/serving-binaries/#package-binary-urls) with a fixed date (e.g., `https://p3m.dev/cran/__linux__/noble/2026-04-15`). The team chooses when to bump the snapshot, the same way they choose when to bump the Posit product version.

Bakery's `r.run_install_packages` macro takes the repo URL through its `_os` argument indirectly (it computes the URL from the OS codename). To pin to a snapshot, either bypass the macro and write the `install.packages` RUN command directly with the snapshot URL, or pass a custom `_os` dict whose `Codename` includes the date suffix.

The Python install path has the same issue: pip resolves from PyPI's current state at build time. Pin via a constraints file or a private mirror.

### `r.run_install_packages` with more than one R version

The example resolves to a single R version because the `dependencyConstraints` block uses bare `latest: true`. If a team adds `count: 2` to install two R minor versions, the `r.run_install_packages` macro in Bakery will emit one `RUN install.packages ... && rm -f /tmp/r-packages.txt` per version. The second `RUN` has nothing to read because the first deleted the file.

Workarounds: write the per-version install loop directly in the template (skip the macro), or open an issue against bakery to expose the `clean` parameter on `run_install_packages`. Python has the same caveat with `count: 2`, but `python.run_install_packages` does expose `clean`, so passing `clean=False` plus a manual `RUN rm` works there.

### Tag your images with the team's own version, not just the Posit product version

This example tags each image with the Posit product version it extends (`workbench:2026.04.0`). That is the cleanest mapping for a team that rebuilds when Posit ships and never customizes mid-cycle. If your team adds packages or system deps between Posit releases, append a team-side suffix (`workbench:2026.04.0-2`, or `:2026.04.0+team-r3`). Bakery does not have an opinion here. Set `name:` under `versions:` to whatever the team's tagging scheme requires.

## Template variables

| Variable | Description | Example |
|:---------|:------------|:--------|
| `{{ Image.Version }}` | Posit product version this image extends | `"2026.04.0"` |
| `{{ Path.Version }}` | Path to the version directory | `"workbench/2026.04.0"` |
| `{{ Dependencies.R }}` | Resolved R versions | `["4.5.3"]` |
| `{{ Dependencies.python }}` | Resolved Python versions | `["3.14.4"]` |

See [TEMPLATING.md](https://github.com/posit-dev/images-shared/blob/main/posit-bakery/TEMPLATING.md) for the full reference.

## Related examples

- [extending/](..): the standalone Containerfile siblings of this example. Start there if you only need to customize one image.
- [bakery/01-basic-image](../../bakery/01-basic-image/): the simplest possible Bakery project, on a stock OS base. Useful for understanding the templating mechanics this example builds on.
- [bakery/05-images-with-managed-dependencies](../../bakery/05-images-with-managed-dependencies/): the `dependencyConstraints` mechanism used here to resolve R and Python.

[DependencyConstraint]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#dependencyconstraint
[ImageVersion]: https://github.com/posit-dev/images-shared/blob/main/posit-bakery/CONFIGURATION.md#imageversion
