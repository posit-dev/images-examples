# Extending Posit container images

Posit product images are published in two variants:

- **Standard** (`std`) — Includes pre-installed versions of Python, R, and Quarto
- **Minimal** (`min`) — A lightweight base image without pre-installed languages, intended to be extended

Product images are available on [Docker Hub](https://hub.docker.com/u/posit):

- [`posit/connect`](https://hub.docker.com/r/posit/connect) — [Posit Connect](https://github.com/posit-dev/images-connect)
- [`posit/connect-content`](https://hub.docker.com/r/posit/connect-content) — [Posit Connect content runtime](https://github.com/posit-dev/images-connect)
- [`posit/package-manager`](https://hub.docker.com/r/posit/package-manager) — [Posit Package Manager](https://github.com/posit-dev/images-package-manager)
- [`posit/workbench`](https://hub.docker.com/r/posit/workbench) — [Posit Workbench](https://github.com/posit-dev/images-workbench)
- [`posit/workbench-session`](https://hub.docker.com/r/posit/workbench-session) — [Posit Workbench session runtime](https://github.com/posit-dev/images-workbench)

> [!NOTE]
> For an alternative approach that uses the Bakery CLI to manage extended images as a project, see the [bakery examples](../bakery/).

## Examples

Examples are organized by product. Within each product folder a `README.md` explains which image to customize for different goals.

| Path | Contents |
|:-----|:---------|
| [common](./common/) | Patterns that apply to any Posit product image |
| [workbench](./workbench/) | Examples for Posit Workbench images — with a guide to choosing the right image |
| [connect](./connect/) | Examples for Posit Connect images — with a guide to choosing the right image |

### Common examples

| Path | Example |
|:-----|:--------|
| [common/ca-certificates](./common/ca-certificates/) | Add a custom CA certificate to the system trust store |
| [common/system-dependencies](./common/system-dependencies/) | Install system libraries required by R or Python packages |

To configure a Python package index, use product admin settings rather than baking a `pip.conf` into the image: [Workbench](https://docs.posit.co/ide/server-pro/admin/python/package_installation.html#setting-a-python-package-index-for-sessions) · [Connect](https://docs.posit.co/connect/admin/python/package-management/#python-package-repositories)
