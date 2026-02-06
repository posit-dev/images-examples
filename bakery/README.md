# Managing Container Images with Bakery

Bakery is Posit's Jinja2-based templating system for managing container images. These examples demonstrate how to use Bakery to generate version-specific container build files from templates.

## Examples

| Path | Example |
|:-----|:--------|
| [01-basic-image](./01-basic-image/) | Single image with one version - the simplest Bakery use case |
| [02-multiple-images-and-versions](./02-multiple-images-and-versions/) | Multiple image families with multiple versions each (Ubuntu and Rocky Linux) |
| [03-image-variants](./03-image-variants/) | Build minimal and standard variants of the same image from a single template |
| [04-image-oses](./04-image-oses/) | Build images for multiple operating systems combined with variants |
| [05-images-with-managed-dependencies](./05-images-with-managed-dependencies/) | Automatic version management of R, Python, and Quarto using dependency constraints |
| [06-matrix-images](./06-matrix-images/) | Build multiple image variants from dependency combinations using Docker build arguments |
