# Posit Container Image Examples

> [!NOTE]
> These images are in preview as Posit migrates container images from [rstudio/rstudio-docker-products](https://github.com/rstudio/rstudio-docker-products). The existing images remain supported.

This repository contains examples for working with [Posit Container Images](https://github.com/posit-dev/images). If you want to run pre-built images, see the Quick Start guides in each product's repository ([Connect](https://github.com/posit-dev/images-connect/blob/main/connect/README.md), [Package Manager](https://github.com/posit-dev/images-package-manager/blob/main/package-manager/README.md), [Workbench](https://github.com/posit-dev/images-workbench/blob/main/workbench/README.md)). The examples here are for users who want to build custom images or extend the base images.

## Examples

| Path | Example |
|:-----|:--------|
| [bakery](./bakery/) | Examples of managing images with [bakery](https://github.com/posit-dev/images-shared/tree/main/posit-bakery) |
| [extending](./extending/) | Examples of extending Posit Container images |

## When to Use These Examples

| Goal | Where to go |
|------|-------------|
| Run a pre-built Posit product image | Product repos: [Connect](https://github.com/posit-dev/images-connect/blob/main/connect/README.md), [Package Manager](https://github.com/posit-dev/images-package-manager/blob/main/package-manager/README.md), [Workbench](https://github.com/posit-dev/images-workbench/blob/main/workbench/README.md) |
| Add R/Python packages or system libraries to a product image | [Extending examples](./extending/) |
| Learn the Bakery build system for managing container images | [Bakery examples](./bakery/) |
| Deploy on Kubernetes with Helm | Product repos or [Posit Helm charts](https://docs.posit.co/helm/) |

## Related repositories

These examples use Posit product images ([Connect](https://github.com/posit-dev/images-connect), [Package Manager](https://github.com/posit-dev/images-package-manager), [Workbench](https://github.com/posit-dev/images-workbench)) and shared build tooling from [images-shared](https://github.com/posit-dev/images-shared). For an overview of the full ecosystem, see [Posit Container Images](https://github.com/posit-dev/images).

## Share your feedback

We invite you to join us on [GitHub Discussions](https://github.com/posit-dev/images/discussions) to ask questions and share feedback.

## Issues

If you encounter any issues or have any questions, please [open an issue](https://github.com/posit-dev/images-examples/issues). We appreciate your feedback.

## Code of conduct

We expect all contributors to adhere to the project's [Code of Conduct](CODE_OF_CONDUCT.md) and create a positive and inclusive community.

## License

Posit Container Images and associated tooling are licensed under the [MIT License](LICENSE.md).
