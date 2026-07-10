# K0rdent Devcontainer

This directory configures the development container environment for working with K0rdent.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
- [Devcontainer Structure](#devcontainer-structure)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Overview

The `.devcontainer` directory provides a ready-to-use development environment using Docker Compose and VS Code Dev Containers. It includes all necessary tools and configuration to work with K0rdent in a reproducible, isolated container.

---

## Prerequisites

- [Docker](https://www.docker.com/)
- [Visual Studio Code](https://code.visualstudio.com/)
- [Remote - Containers Extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

---

## Setup Instructions

1. Open this repository in VS Code.
2. You can now use the container for K0rdent development and testing.

    ```shell
    k0sctl kubeconfig \
        --address https://127.0.0.1:6443 \
        --config ./hack/cluster.yaml \
        >./hack/kubeconfig.yaml
    ```

See [README](../README.md#3-install-kcm) for to continue with deployment.

---

## Devcontainer Structure

- `Dockerfile`: Builds the devcontainer image with all required tools.
- `devcontainer.json`: Main configuration for the devcontainer, referencing `compose-dev.yaml`.
- `compose-dev.yaml`: Docker Compose file defining the `kcm` service, mounting the workspace, setting up volumes (including CRDs and SSH keys), and exposing ports (10022, 30443, 443, 6443, 80).

---

## Troubleshooting

- **Devcontainer Fails to Start**: Ensure Docker is running and you have the VS Code Remote - Containers extension installed.
- **SSH Issues**: Make sure the `authorized_keys` file is present and correct.
- **Volume Mount Problems**: Check that the workspace and required files are accessible to Docker.

---

## License

Copyright (c) 2025 Schubert Anselme <schubert@anselm.es>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <https://www.gnu.org/licenses/>.
