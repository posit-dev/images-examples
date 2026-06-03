# Extending Posit Workbench images

Workbench deployments on Kubernetes use several images that serve different roles. The right image to customize depends on what you want to change.

## Which image do I customize?

| I want to… | Customize | Example |
|:-----------|:----------|:--------|
| Add R or Python packages available in every user session | [`workbench-session`](https://hub.docker.com/r/posit/workbench-session) | [session/r-python-packages](./session/r-python-packages/) |
| Add system libraries that session packages need | [`workbench-session`](https://hub.docker.com/r/posit/workbench-session) | [session/system-dependencies](./session/system-dependencies/) |
| Change Workbench server configuration | [`workbench`](https://hub.docker.com/r/posit/workbench) (Minimal) | [server/config](./server/config/) |
| Install additional languages on the Workbench server | [`workbench`](https://hub.docker.com/r/posit/workbench) (Minimal) | [server/python](./server/python/) |
| Install Posit Pro Drivers on the Workbench server | [`workbench`](https://hub.docker.com/r/posit/workbench) (Minimal) | [server/pro-drivers](./server/pro-drivers/) |
| Pre-install VS Code extensions | [`workbench`](https://hub.docker.com/r/posit/workbench) (Standard) | [server/vs-code-extensions](./server/vs-code-extensions/) |
| Bundle session components into a self-contained session image | [`workbench-session-init`](https://hub.docker.com/r/posit/workbench-session-init) + `workbench-session` | [session-init](./session-init/) |
| Run a newer Positron version than the Workbench server ships | [`workbench-positron-init`](https://hub.docker.com/r/posit/workbench-positron-init) | [Helm config](#upgrading-positron-independently) |
| Upgrade the Workbench server version | `workbench` + `workbench-session-init` (keep in sync) | — |

## How the images fit together

**For local Docker deployments,** you only need the `workbench` image. Sessions run as processes inside the server container.

**For Kubernetes deployments,** four images work together:

- **`workbench`** — the Workbench server and Job Launcher. Runs as the main server pod.
- **`workbench-session`** — the runtime environment for interactive user sessions (RStudio, VS Code, Positron, Jupyter). The Job Launcher schedules these as session pods. This is the image that user sessions run in, so it is the most common customization target when you want to add R/Python packages.
- **`workbench-session-init`** — an init container that supplies the Workbench session components to session pods by copying them into a shared volume. Its version must always match the `workbench` server version.
- **`workbench-positron-init`** — an init container that supplies the Positron IDE to session pods. It is managed separately from the Workbench release cycle, so you can upgrade Positron without upgrading Workbench.

## Examples

### Server examples

These examples build on the `workbench` Minimal image (`-min`), adding configuration, languages, or drivers to the Workbench server.

| Path | Example |
|:-----|:--------|
| [server/config](./server/config/) | Apply a custom Workbench server configuration file |
| [server/python](./server/python/) | Install specific versions of Python on a minimal Workbench image |
| [server/pro-drivers](./server/pro-drivers/) | Install the Posit Pro Drivers (ODBC drivers) |
| [server/system-dependencies](./server/system-dependencies/) | Install system dependencies required for additional libraries |
| [server/vs-code-extensions](./server/vs-code-extensions/) | Pre-install a list of VS Code extensions (Standard image) |

### Session examples

These examples build on the `workbench-session` image, adding packages or system libraries that user sessions need.

| Path | Example |
|:-----|:--------|
| [session/r-python-packages](./session/r-python-packages/) | Pre-install R and Python packages in a session image |
| [session/system-dependencies](./session/system-dependencies/) | Install system dependencies required by session packages |

### Session-init example

| Path | Example |
|:-----|:--------|
| [session-init](./session-init/) | Bundle session components into a self-contained session image |

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
