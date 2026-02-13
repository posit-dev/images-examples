# Extending Posit Container Images

Posit product images are published in two variants:
- **Standard** (`std`) — Includes pre-installed versions of Python, R, and Quarto
- **Minimal** (`min`) — A lightweight base image without pre-installed languages, intended to be extended

These examples use Minimal images as base images. Product images are available on [Docker Hub](https://hub.docker.com/u/posit):
- [`posit/connect`](https://hub.docker.com/r/posit/connect) — [Posit Connect](https://github.com/posit-dev/images-connect)
- [`posit/package-manager`](https://hub.docker.com/r/posit/package-manager) — [Posit Package Manager](https://github.com/posit-dev/images-package-manager)
- [`posit/workbench`](https://hub.docker.com/r/posit/workbench) — [Posit Workbench](https://github.com/posit-dev/images-workbench)

> [!TIP]
> For an alternative approach using the Bakery CLI to manage extended images as a project, see the [bakery examples](../bakery/).

## Examples

| Path | Example |
|:-----|:--------|
| [ca-certificates](./ca-certificates/) | Add a custom CA certificate to the system trust store |
| [pip-conf](./pip-conf/) | Add a custom `pip.conf` file to specify global pip settings |
| [python](./python/) | Install specific versions of Python on a minimal product image<br/>Install a list of python packages in each python version |
| [R](./R/) | Install specific versions of R on a minimal product image<br/>Install a list of R packages in each R version |
| [system-dependencies](./system-dependencies/) | Install system dependencies required for additional libraries |
| [vs-code-extensions](./vs-code-extensions/) | Pre-install a list of VS Code extensions |
