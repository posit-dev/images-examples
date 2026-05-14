# Extending Posit container images

Posit publishes product images in two variants:
- **Standard** (`std`): Includes pre-installed versions of Python, R, and Quarto.
- **Minimal** (`min`): A lightweight base image without pre-installed languages. Extend it to add the languages and packages you need.

These examples build on the Minimal images. Product images are available on [Docker Hub](https://hub.docker.com/u/posit):
- [`posit/connect`](https://hub.docker.com/r/posit/connect): [Posit Connect](https://github.com/posit-dev/images-connect)
- [`posit/package-manager`](https://hub.docker.com/r/posit/package-manager): [Posit Package Manager](https://github.com/posit-dev/images-package-manager)
- [`posit/workbench`](https://hub.docker.com/r/posit/workbench): [Posit Workbench](https://github.com/posit-dev/images-workbench)

> [!NOTE]
> For an alternative approach that uses the Bakery CLI to manage extended images as a project, see the [bakery examples](../bakery/).

## Examples

Each example extends a specific Posit product image. The patterns shown in any example apply to the other products as well.

| Path | Base product | Example |
|:-----|:-------------|:--------|
| [ca-certificates](./ca-certificates/) | Posit Package Manager | Add a custom CA certificate to the system trust store |
| [pip-conf](./pip-conf/) | Posit Connect | Add a custom `pip.conf` file to specify global pip settings |
| [pro-drivers](./pro-drivers/) | Posit Workbench | Install the Posit Pro Drivers (ODBC drivers) on a minimal product image |
| [python](./python/) | Posit Workbench | Install specific versions of Python on a minimal product image<br/>Install a list of Python packages in each Python version |
| [quarto](./quarto/) | Posit Connect | Install Quarto and TinyTeX on a minimal product image |
| [R](./R/) | Posit Connect | Install specific versions of R on a minimal product image<br/>Install a list of R packages in each R version |
| [system-dependencies](./system-dependencies/) | Posit Workbench | Install system dependencies required for additional libraries |
| [vs-code-extensions](./vs-code-extensions/) | Posit Workbench (Standard) | Pre-install a list of VS Code extensions |
