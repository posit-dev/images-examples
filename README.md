# Posit container image examples

This repository contains examples for working with [Posit Container Images](https://github.com/posit-dev/images). To run pre-built images, see the quick start guides in each product repository: [Posit Connect](https://github.com/posit-dev/images-connect/blob/main/connect/README.md), [Posit Package Manager](https://github.com/posit-dev/images-package-manager/blob/main/package-manager/README.md), and [Posit Workbench](https://github.com/posit-dev/images-workbench/blob/main/workbench/README.md). The examples here are for users who want to build custom images or extend the base images.

## Extending vs Bakery

Are you customizing one image or managing many?

- [Extending](./extending/): Start from a pre-built Posit image and add what you need in a standard Dockerfile. Use this approach to add R and Python packages, system libraries, or custom configuration to a single product image. No special tooling required.

- [Bakery](./bakery/): The Posit [templating system](https://github.com/posit-dev/images-shared/tree/main/posit-bakery) for managing matrices of container images across multiple R versions, Python versions, OS variants, and product versions. Use Bakery to maintain a fleet of custom images and rebuild them consistently. Posit uses the same tool to build the official product images.

Looking for something else?

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

## Contributing

Before your first commit, install the repository's [`pre-commit`](https://pre-commit.com/) hooks so changes are checked locally before they hit CI:

```shell
pre-commit install
```

## Code of conduct

We expect all contributors to adhere to the project's [Code of Conduct](CODE_OF_CONDUCT.md) and create a positive and inclusive community.

## License

Posit Container Images and associated tooling are licensed under the [MIT License](LICENSE.md).
