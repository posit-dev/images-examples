# Extending Posit container images

Posit product images are published in two variants:

- **Standard** (`std`): Includes pre-installed versions of Python, R, and Quarto
- **Minimal** (`min`): A lightweight base image without pre-installed languages, intended to be extended

Product images are available on [Docker Hub](https://hub.docker.com/u/posit):

- [`posit/connect`](https://hub.docker.com/r/posit/connect): [Posit Connect](https://github.com/posit-dev/images-connect)
- [`posit/connect-content`](https://hub.docker.com/r/posit/connect-content): [Posit Connect content runtime](https://github.com/posit-dev/images-connect)
- [`posit/package-manager`](https://hub.docker.com/r/posit/package-manager): [Posit Package Manager](https://github.com/posit-dev/images-package-manager)
- [`posit/workbench`](https://hub.docker.com/r/posit/workbench): [Posit Workbench](https://github.com/posit-dev/images-workbench)
- [`posit/workbench-session`](https://hub.docker.com/r/posit/workbench-session): [Posit Workbench session runtime](https://github.com/posit-dev/images-workbench)

## Built with Bakery

These use [Bakery](../bakery/) to template and manage image definitions, typically across several Posit products or several versions at once. The rendered Containerfiles are committed alongside the templates, so a build does not require Bakery: `docker build` against the rendered file works the same way the static examples do.

| Path | Example |
|:-----|:--------|
| [posit-team](./posit-team/) | Manage a fleet of customized Workbench, Connect, and Package Manager images with shared R and Python pins |

## Static image definitions

Examples are organized by product. Within each product folder, a `README.md` explains which image to customize for different goals.

| Path | Base product | Example |
|:-----|:-------------|:--------|
| [common/ca-certificates](./common/ca-certificates/) | Any | Add a custom CA certificate to the system trust store |
| [pip-conf](./pip-conf/) | Posit Connect | Add a custom `pip.conf` file to specify global pip settings |
| [pro-drivers](./pro-drivers/) | Posit Workbench | Install the Posit Pro Drivers (ODBC drivers) on a minimal product image |
| [common/python](./common/python/) | Any | Install specific versions of Python on a minimal product image<br/>Install a list of Python packages in each Python version |
| [quarto](./quarto/) | Posit Connect | Install Quarto and TinyTeX on a minimal product image |
| [common/R](./common/R/) | Any | Install specific versions of R on a minimal product image<br/>Install a list of R packages in each R version |
| [common/system-dependencies](./common/system-dependencies/) | Any | Install system dependencies required for additional libraries |
| [vs-code-extensions](./vs-code-extensions/) | Posit Workbench (Standard) | Pre-install a list of VS Code extensions |

To configure a Python package index, use product admin settings rather than baking a `pip.conf` into the image: [Workbench](https://docs.posit.co/ide/server-pro/admin/python/package_installation.html#setting-a-python-package-index-for-sessions) · [Connect](https://docs.posit.co/connect/admin/python/package-management/#python-package-repositories)
