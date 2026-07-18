# K0rdent Sandbox

This repository is a sandbox for day-2 operations with KCM (k0rdent-enterprise): installing provider, service, and cluster templates, configuring management services, and deploying MaaS-based child clusters on an existing management cluster.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [1. Install KCM](#1-install-kcm)
  - [2. Register the Helm Repository](#2-register-the-helm-repository)
  - [3. Install the Provider Template](#3-install-the-provider-template)
  - [4. Install the Service Templates](#4-install-the-service-templates)
  - [5. Install the Cluster Templates](#5-install-the-cluster-templates)
  - [6. Create the Vault Credential Secret](#6-create-the-vault-credential-secret)
  - [7. Configure Management](#7-configure-management)
  - [8. Deploy Cluster Services](#8-deploy-cluster-services)
  - [9. Deploy Child Clusters](#9-deploy-child-clusters)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)
- [Additional Documentation](#additional-documentation)
- [License](#license)

---

## Overview

This guide walks through setting up KCM on a management cluster and deploying child clusters with the Cluster API MaaS provider. It installs the k0rdent-enterprise Helm chart, registers an OCI Helm repository for template charts, installs provider/service/cluster templates, wires up management services (CA issuer, external-dns, external-secrets, Gateway API with Envoy, and Vault), and finally deploys MaaS child clusters with either a standalone control plane (`mscp`) or a hosted control plane (`mhcp`).

Bringing up the management cluster itself is out of scope for this repository — it is covered in the devbox repository. The instructions below assume you already have a running cluster and a kubeconfig pointing at it.

---

## Prerequisites

Before starting, ensure you have the following:

- A running Kubernetes management cluster and a kubeconfig with admin access
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/)
- Network access to `registry.mirantis.com` and `ghcr.io` for pulling OCI charts
- A reachable [MaaS](https://maas.io/) environment with credentials for provisioning machines
- A Vault AppRole secret ID for the external-secrets integration

> [!NOTE]
> Several manifests contain environment-specific values (MaaS endpoint, control plane endpoint IP, DNS domain). Review and replace them with values for your environment before applying, as called out in the steps below.

---

## Setup Instructions

### 1. Install KCM

Install the k0rdent-enterprise chart into the `kcm-system` namespace:

```shell
helm install kcm \
  oci://registry.mirantis.com/k0rdent-enterprise/charts/k0rdent-enterprise \
  --version 1.4.0 \
  --namespace kcm-system \
  --create-namespace
```

### 2. Register the Helm Repository

Register the OCI Helm repository that serves the template charts used in the following steps:

```shell
kubectl apply -f deployment/repo.yaml
```

### 3. Install the Provider Template

Install the Cluster API MaaS provider template:

```shell
kubectl apply -f deployment/template/provider
```

### 4. Install the Service Templates

Service template charts are delivered through [kgst](https://github.com/labsonline) (k0rdent guided service templates). Install the CNI, CSI, and core service charts:

```shell
# cni template
helm upgrade --install --namespace kcm-system cni-1-0-7 oci://ghcr.io/labsonline/charts/kgst --set kgst.chart=cni:1.0.7

# csi template
helm upgrade --install --namespace kcm-system csi-1-0-7 oci://ghcr.io/labsonline/charts/kgst --set kgst.chart=csi:1.0.7

# core templates
helm upgrade --install --namespace kcm-system ca-1-0-4 oci://ghcr.io/labsonline/charts/kgst --set kgst.chart=ca:1.0.4
helm upgrade --install --namespace kcm-system edns-1-19-0 oci://ghcr.io/labsonline/charts/kgst --set kgst.chart=edns:1.19.0
helm upgrade --install --namespace kcm-system eso-2-5-0 oci://ghcr.io/labsonline/charts/kgst --set kgst.chart=eso:2.5.0
helm upgrade --install --namespace kcm-system gateway-1-0-3 oci://ghcr.io/labsonline/charts/kgst --set kgst.chart=gateway:1.0.3
helm upgrade --install --namespace kcm-system gwapi-1-0-3 oci://ghcr.io/labsonline/charts/kgst --set kgst.chart=gwapi:1.0.3
```

Then apply the service template chains that group these templates for delivery to child clusters:

```shell
kubectl apply -f deployment/template/service
```

### 5. Install the Cluster Templates

Install the MaaS cluster templates (hosted control plane, standalone control plane) and the cluster template chain:

```shell
kubectl apply -f deployment/template/cluster/mhcp.yaml
kubectl apply -f deployment/template/cluster/mscp.yaml
kubectl apply -f deployment/template/cluster/maas.yaml
```

### 6. Create the Vault Credential Secret

The external-secrets management service authenticates to Vault with an AppRole secret ID. Copy [example.env](example.env) to `.env`, set `VAULT_KCM_SECRET_ID` to your Vault AppRole secret ID, then create the secret:

```shell
cp example.env .env
source .env

kubectl create namespace sre
kubectl -n sre create secret generic external-secrets-vault-credential --from-literal secret-id=$VAULT_KCM_SECRET_ID
```

### 7. Configure Management

Apply the management services (CA issuer, external-dns, external-secrets, Envoy, Gateway, Vault), the `Management` object, and the `AccessManagement` rules:

```shell
kubectl apply -f deployment/mgmt
kubectl apply -f deployment/mgmt.yaml
kubectl apply -f deployment/access.yaml
```

> [!IMPORTANT]
> [deployment/mgmt.yaml](deployment/mgmt.yaml) sets `MAAS_ENDPOINT` for the MaaS provider. Replace it with the URL of your MaaS environment before applying.

### 8. Deploy Cluster Services

Apply the `MultiClusterService` resources that deliver core services, DNS, gateway, issuer, and secret management to matching child clusters:

```shell
kubectl apply -f deployment/service
```

### 9. Deploy Child Clusters

Child cluster manifests target the `default` namespace, which must be labelled to opt in to root CA and cloud-init delivery:

```shell
kubectl label namespace default rootca=enabled      # enable rootca for default namespace
kubectl label namespace default cloudinit=enabled   # enable cloudinit for default namespace
```

Then deploy one or both of the MaaS child clusters:

```shell
kubectl apply -f deployment/cluster/mscp.yaml       # MaaS standalone control plane
kubectl apply -f deployment/cluster/mhcp.yaml       # MaaS hosted control plane
```

> [!IMPORTANT]
> Review [deployment/cluster/mscp.yaml](deployment/cluster/mscp.yaml) and [deployment/cluster/mhcp.yaml](deployment/cluster/mhcp.yaml) first: `controlPlaneEndpointIP`, the MaaS `dnsDomain`, and machine sizing are environment-specific and must match your MaaS setup.

---

## Cleanup

Delete the child clusters first and wait for their machines to be released before removing anything else:

```shell
kubectl delete -f deployment/cluster/mscp.yaml
kubectl delete -f deployment/cluster/mhcp.yaml
kubectl wait --for=delete clusterdeployment --all --namespace default --timeout=1800s
```

Then remove the cluster services, management configuration, and templates:

```shell
kubectl delete -f deployment/service
kubectl delete -f deployment/access.yaml
kubectl delete -f deployment/mgmt
kubectl delete -f deployment/template/cluster/maas.yaml
kubectl delete -f deployment/template/cluster/mscp.yaml
kubectl delete -f deployment/template/cluster/mhcp.yaml
kubectl delete -f deployment/template/service
kubectl delete -f deployment/template/provider
kubectl delete -f deployment/repo.yaml
kubectl -n sre delete secret external-secrets-vault-credential
kubectl delete namespace sre
```

Finally, uninstall the Helm releases and remove local artifacts:

```shell
helm --namespace kcm-system uninstall \
  cni-1-0-7 csi-1-0-7 ca-1-0-4 edns-1-19-0 eso-2-5-0 gateway-1-0-3 gwapi-1-0-3
helm --namespace kcm-system uninstall kcm

rm -f .env kubeconfig.*.yaml
```

> [!CAUTION]
> Deleting a `ClusterDeployment` deprovisions the corresponding MaaS machines. Make sure nothing you need is running on them before cleaning up.

---

## Troubleshooting

- **`Management/kcm` not ready**: Check the KCM controller logs in the `kcm-system` namespace (`kubectl -n kcm-system logs deploy/kcm-controller-manager`) and verify the referenced release and provider templates exist.
- **Child cluster stuck provisioning**: Check the MaaS provider controller logs (`kubectl -n kcm-system logs deploy/capmaas-controller-manager`) and confirm the `MAAS_ENDPOINT` and MaaS credentials are correct and reachable.
- **Services not delivered to a child cluster**: Verify the `AccessManagement` rules in [deployment/access.yaml](deployment/access.yaml) target the cluster's namespace and that the namespace carries the `rootca=enabled` and `cloudinit=enabled` labels.
- **Stale DNS after recreating a cluster**: The MaaS provider generates the control plane host suffix, so any `DNSEndpoint` pinned to the old hostname (see the note in [deployment/cluster/mscp.yaml](deployment/cluster/mscp.yaml)) must be updated after the cluster is recreated.

---

## Additional Documentation

| Document                                           | Description                                                       |
| -------------------------------------------------- | ----------------------------------------------------------------- |
| [docs/aws.md](docs/aws.md)                         | Preparing AWS credentials for the Cluster API AWS provider        |
| [docs/cilium.md](docs/cilium.md)                   | Kubernetes zero-trust networking with Cilium                      |
| [docs/gatekeeper.md](docs/gatekeeper.md)           | Gatekeeper and Cilium: admission control plus network enforcement |
| [.devcontainer/README.md](.devcontainer/README.md) | Development container environment for working with K0rdent        |

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
