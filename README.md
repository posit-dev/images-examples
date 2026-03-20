# Posit Container Image Examples

> [!NOTE]
> These images are in preview as Posit migrates container images from [rstudio/rstudio-docker-products](https://github.com/rstudio/rstudio-docker-products). The existing images remain supported.

This repository contains examples for working with [Posit Container Images](https://github.com/posit-dev/images). If you want to run pre-built images, see the Quick Start guides in each product's repository ([Posit Connect](https://github.com/posit-dev/images-connect/blob/main/connect/README.md), [Posit Package Manager](https://github.com/posit-dev/images-package-manager/blob/main/package-manager/README.md), [Posit Workbench](https://github.com/posit-dev/images-workbench/blob/main/workbench/README.md)). The examples here are for users who want to build custom images or extend the base images.

## Extending vs Bakery

**Are you customizing one image or managing many?**

- **[Extending](./extending/)** — Start from a pre-built Posit image and add what you need in a standard Dockerfile. Use this to add R/Python packages, system libraries, or custom configuration to a single product image. No special tooling required.

- **[Bakery](./bakery/)** — Posit's [templating system](https://github.com/posit-dev/images-shared/tree/main/posit-bakery) for managing matrices of container images across multiple R versions, Python versions, OS variants, and product versions. Use this if you maintain a fleet of custom images and need to rebuild them consistently. This is the same tool Posit uses to build the official product images.

**Looking for something else?**

| Goal | Where to go |
|------|-------------|
| Run a pre-built Posit product image | [Connect](https://github.com/posit-dev/images-connect/blob/main/connect/README.md), [Package Manager](https://github.com/posit-dev/images-package-manager/blob/main/package-manager/README.md), [Workbench](https://github.com/posit-dev/images-workbench/blob/main/workbench/README.md) |
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
