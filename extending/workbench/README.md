# Extending Posit Workbench images

Each Workbench image serves a different role. The right image to customize depends on what you want to change and your deployment architecture.

For local Docker, all sessions run inside the `workbench` container, so session-level customization applies to the `workbench` image. When running Workbench with Kubernetes, reference the table below to identify the target image for customization.

## Which image do I customize?

| I want to… | Customize | Example |
|:-----------|:----------|:--------|
| Add R or Python packages to sessions | [`workbench-session`](https://hub.docker.com/r/posit/workbench-session) | [session/r-python-packages](./session/r-python-packages/) |
| Add system libraries that session packages need | [`workbench-session`](https://hub.docker.com/r/posit/workbench-session) | [session/system-dependencies](./session/system-dependencies/) |
| Bundle session components into a self-contained session image | [`workbench-session-init`](https://hub.docker.com/r/posit/workbench-session-init) + `workbench-session` | [session-init](./session-init/) |
| Run a newer Positron version than the Workbench server ships | [`workbench-positron-init`](https://hub.docker.com/r/posit/workbench-positron-init) | [Helm config](#upgrading-positron-independently) |
| Upgrade the Workbench server version | `workbench` + `workbench-session-init` (keep in sync) | — |

## How the images fit together

**For local Docker deployments,** you only need the `workbench` image. Sessions run as processes inside the server container. Examples: [server/config](./server/config/), [server/pro-drivers](./server/pro-drivers/), [server/vs-code-extensions](./server/vs-code-extensions/).

**For Kubernetes deployments,** four images work together:

- **`workbench`** — the Workbench server and Job Launcher. Runs as the main server pod.
- **`workbench-session`** — the runtime environment for interactive user sessions (RStudio, VS Code, Positron, Jupyter). The Job Launcher schedules these as session pods. This is the image that user sessions run in, so it is the most common customization target when you want to add R/Python packages.
- **`workbench-session-init`** — an init container that supplies the Workbench session components to session pods by copying them into a shared volume. Its version must always match the `workbench` server version.
- **`workbench-positron-init`** — an init container that supplies the Positron IDE to session pods. It is managed separately from the Workbench release cycle, so you can upgrade Positron without upgrading Workbench.

## Upgrading Positron independently

To run a Positron version that differs from what the Workbench server ships, set `components.positron.version` in your Helm values. The `workbench-positron-init` init container fetches the specified version and supplies it to session pods at runtime. No custom image build is required.

```yaml
components:
  positron:
    version: "2026.05.1-2"
    image:
      repository: "ghcr.io/posit-dev/workbench-positron-init"
```

See the [rstudio-workbench Helm chart documentation](https://docs.posit.co/helm/charts/rstudio-workbench/README.html) for the full list of available values.
