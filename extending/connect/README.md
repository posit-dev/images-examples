# Extending Posit Connect images

Each Connect image serves a different role. The right image to customize depends on what you want to change and your deployment architecture.

For local Docker, all content runs inside the `connect` container, so content-level customization applies to the `connect` image. When running Connect with off-host execution (OHE), reference the table below to identify the target image for customization. See [Architecture Overview](https://docs.posit.co/connect/admin/appendix/off-host/arch-overview/) for a description of local execution and off-host execution (OHE).

## Which image do I customize?

| I want to… | Customize | Example |
|:-----------|:----------|:--------|
| Install specific versions of R or Python | [`connect-content`](https://hub.docker.com/r/posit/connect-content) | [common/R](../common/R/) · [common/python](../common/python/) |
| Add system libraries that content packages need | [`connect-content`](https://hub.docker.com/r/posit/connect-content) | [common/system-dependencies](../common/system-dependencies/) |
| Upgrade the Connect server version | `connect` + `connect-content-init` (keep in sync) | — |
| Add custom runtime components for off-host execution | [`connect-content-init`](https://hub.docker.com/r/posit/connect-content-init) | [Custom container images](https://docs.posit.co/helm/examples/connect/container-images/custom-images.html) |

## How the images fit together

**For local Docker deployments,** you only need the `connect` image. Content runs as processes inside the Connect server container using the bundled R, Python, and Quarto. To install a specific Quarto version on the server, see [server/quarto](./server/quarto/).

**For Kubernetes deployments with off-host execution (OHE),** three images work together:

- **`connect`** — the Connect server and Job Launcher. Runs as the main server pod.
- **`connect-content`** — the runtime environment for executing published content (Shiny apps, R Markdown, Quarto docs, Plumber APIs, and Python equivalents) in off-host execution pods. The Connect Job Launcher schedules these as content-execution pods. This is the image your content actually runs in, so it is the most common customization target when you want to add R/Python packages.
- **`connect-content-init`** — an init container that supplies the Connect runtime components to content-execution pods by copying them into a shared volume. Its version must always match the `connect` server version.
