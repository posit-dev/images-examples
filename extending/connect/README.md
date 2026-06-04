# Extending Posit Connect images

Connect deployments on Kubernetes use several images that serve different roles. The right image to customize depends on what you want to change.

## Which image do I customize?

| I want to… | Customize | Example |
|:-----------|:----------|:--------|
| Add system libraries that content packages need | [`connect-content`](https://hub.docker.com/r/posit/connect-content) | [common/system-dependencies](../common/system-dependencies/) |
| Install R on the Connect server (for local Docker) | [`connect`](https://hub.docker.com/r/posit/connect) (Minimal) | [server/R](./server/R/) |
| Install Quarto on the Connect server | [`connect`](https://hub.docker.com/r/posit/connect) (Minimal) | [server/quarto](./server/quarto/) |
| Upgrade the Connect server version | `connect` + `connect-content-init` (keep in sync) | — |
| Add custom runtime components for off-host execution | [`connect-content-init`](https://hub.docker.com/r/posit/connect-content-init) | [Custom container images](https://docs.posit.co/helm/examples/connect/container-images/custom-images.html) |

## How the images fit together

**For local Docker deployments,** you only need the `connect` image. Content runs as processes inside the Connect server container using the bundled R, Python, and Quarto.

**For Kubernetes deployments with off-host execution (OHE),** three images work together:

- **`connect`** — the Connect server and Job Launcher. Runs as the main server pod.
- **`connect-content`** — the runtime environment for executing published content (Shiny apps, R Markdown, Quarto docs, Plumber APIs, and Python equivalents) in off-host execution pods. The Connect Job Launcher schedules these as content-execution pods. This is the image your content actually runs in, so it is the most common customization target when you want to add R/Python packages.
- **`connect-content-init`** — an init container that supplies the Connect runtime components to content-execution pods by copying them into a shared volume. Its version must always match the `connect` server version.

`connect-content` images come in two variants:

- **`base`** — Open-source R and Python with system libraries for common packages.
- **`pro`** — Adds Posit Pro Drivers and the `odbc` R package for ODBC database connectivity.

## Examples

### Server examples

These examples build on the `connect` Minimal image (`-min`), adding languages to the Connect server. Use these for local Docker deployments or to add languages to an existing single-server deployment.

| Path | Example |
|:-----|:--------|
| [server/R](./server/R/) | Install specific versions of R on a minimal Connect image |
| [server/quarto](./server/quarto/) | Install Quarto and TinyTeX on a minimal Connect image |

### Content examples

These examples build on the `connect-content` image, adding packages or system libraries that deployed content needs. Use these for Kubernetes off-host execution deployments.

| Path | Example |
|:-----|:--------|
| [common/system-dependencies](../common/system-dependencies/) | Install system dependencies required by content packages |
