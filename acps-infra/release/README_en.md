**[English](README_en.md) | [中文](README.md)**

# ACPs release packaging layer

The new packaging and deployment solution lives in this directory (`app-packaging` / `image-packaging` / `install-packaging`).

## Directory boundaries

| Directory | Responsibility | Main artifacts |
| --- | --- | --- |
| `app-packaging/` | Application release package assembly | app-release kit / thin package |
| `image-packaging/` | image-mode image packages | `*.image.tar.gz` / image manifest |
| `install-packaging/` | Installation package assembly + Ansible | `acps-*-install-*.tar` |
| `lib/` | Shared Python library (such as `runtime_package.py`) | — |

For the concepts see the repository root [`README_en.md`](../README_en.md); for the commands see [acps-docs](../../acps-docs/README_en.md).
