# K0rdent Demo

This repository demonstrates how to set up a Kubernetes cluster using `kind`, deploy KCM (Kubernetes Cluster Manager), and manage workloads.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [0. Prequisites](#0-prequisites)
  - [1. Generate Kind Config File](#1-generate-kind-config-file)
  - [2. Create Management Cluster](#2-create-management-cluster)
  - [3. Install KCM](#3-install-kcm)
  - [4. Deploy Management Workload](#4-deploy-management-workload)
  - [5. Create Workload Cluster](#5-create-workload-cluster)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)

---

## Overview

This guide walks you through setting up a Kubernetes environment using `kind` (Kubernetes in Docker) and deploying workloads with KCM. It includes optional steps for customizing your setup.

---

## Prerequisites

Before starting, ensure you have the following installed on your system:

- [Docker](https://www.docker.com/)
- [Kind](https://kind.sigs.k8s.io/)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/)
- [Flux](https://fluxcd.io/flux/)

---

## Setup Instructions

### 0. Prequisites

Copy config/cert.yaml.example to config/cert.yaml and customize it.

```shell
CF_API_TOKEN="${CF_API_TOKEN}" envsubst < deployment/secret/mgmt/cf.example.env >deployment/secret/mgmt/cf.env
KCM_ADMIN_PWD="${KCM_ADMIN_PWD}" envsubst < deployment/secret/mgmt/kcm.example.env >deployment/secret/mgmt/kcm.env

CF_API_TOKEN="${CF_API_TOKEN}" envsubst < deployment/secret/cf.example.env >deployment/secret/cf.env

PRIVATE_SSH_KEY_B64="${PRIVATE_SSH_KEY_B64}" envsubst < deployment/template/cluster/remote/example.env >deployment/template/cluster/remote/.env

yq '.config' config/cert.yaml -o json >config/pki/openssl.json                # generate openssl config
yq '.ca' config/cert.yaml -o json >config/pki/ca.json                         # generate root ca config
yq '.intermediate' config/cert.yaml -o json >config/pki/intermediate.json     # generate intermediate ca config
yq '.kcm' config/cert.yaml -o json >config/pki/kcm.json                       # generate kcm intermediate ca config

# generate root ca
cfssl genkey -config config/pki/openssl.json -initca config/pki/ca.json | cfssljson -bare config/pki/ca

# generate intermediate ca
cfssl gencert \
  -ca config/pki/ca.pem \
  -ca-key config/pki/ca-key.pem \
  -config config/pki/openssl.json \
  -profile intermediate \
  config/pki/intermediate.json | cfssljson -bare config/pki/intermediate

# generate kcm intermediate ca
cfssl gencert \
  -ca config/pki/ca.pem \
  -ca-key config/pki/ca-key.pem \
  -config config/pki/openssl.json \
  -profile intermediate \
  config/pki/kcm.json | cfssljson -bare config/pki/kcm
```

Use kind to create the management cluster, orbstack or docker-desktop can also be used.

### 1. Generate Kind Config File

Copy the example files to create your `kind` configuration files:

```shell
cp config/kind/kcm.example.yaml config/kind/kcm.yaml
cp config/kind/adopted.example.yaml config/kind/adopted.yaml
```

### 2. Create Management Cluster

Create a management cluster using the generated configuration:

```shell
grep -qa kcm <(kind get clusters) >/dev/null 2>&1 || kind create cluster --config config/kind/kcm.yaml

# TODO: install msr
```

### 3. Install KCM

Deploy KCM using Helm:

```shell
# TODO: install kcm

# wait for KCM to be ready
kubectl wait --for condition=Ready --namespace kcm-system --timeout 720s Management/kcm
```

### 4. Deploy Management Workload

Deploy your management workload using `kustomize`:

```shell
kubectl apply -f <(kustomize build deployment)
```

### 5. Create Workload Cluster

Update `deployment/kustomization.yaml` to include the desired cluster by uncommenting the relevant lines. For example:

```yaml
resources:
  - adopted
  - dev-docker.yaml
  # - dev-openstack.yaml
  # - dev-remote.yaml
  # - dev-vsphere.yaml
```

Create a kind cluster for adoption:

```shell
# create kind cluster
grep -qa adopted <(kind get clusters) >/dev/null 2>&1 || kind create cluster --config config/kind/adopted.yaml

# add secrets
kind get kubeconfig --name adopted >hack/kind-adopted-kubeconfig.yaml
kubectl create namespace cert-manager --dry-run=client -o yaml | kubectl --kubeconfig hack/kind-adopted-kubeconfig.yaml apply -f -
kubectl --kubeconfig hack/kind-adopted-kubeconfig.yaml apply -f <(kustomize build --load-restrictor=LoadRestrictionsNone deployment/secret)

# create credential
BASE64_KUBECONFIG="$(cat hack/kind-adopted-kubeconfig.yaml | base64 -w0)"
BASE64_KUBECONFIG="${BASE64_KUBECONFIG}"  envsubst < deployment/cluster/adopted/example.env >deployment/cluster/adopted/.env
```

Create a workload cluster:

```shell
kustomize build deployment/cluster | kubectl apply -f -

# Export the kubeconfig from the secret to hack/dev-docker-kubeconfig.yaml
kubectl get secret dev-docker-kubeconfig -o jsonpath='{.data.value}' | base64 --decode > hack/dev-docker-kubeconfig.yaml

# add secrets
kubectl create namespace cert-manager --dry-run=client -o yaml | kubectl --kubeconfig hack/dev-docker-kubeconfig.yaml apply -f -
kubectl --kubeconfig hack/dev-docker-kubeconfig.yaml apply -f <(kustomize build --load-restrictor=LoadRestrictionsNone deployment/secret)
```

## Cleanup

To clean up the management cluster and resources, run:

```shell
kubectl delete -f <(kustomize build deployment/cluster)

kind delete cluster --name adopted
kind delete cluster --name kcm

rm -f \
  "config/kind/adopted.yaml" \
  "config/kind/kcm.yaml" \
  "deployment/cluster/adopted/.env" \
  "deployment/secret/.env" \
  "deployment/secret/mgmt/cf.env" \
  "deployment/secret/mgmt/kcm.env" \
  "deployment/template/cluster/remote/.env" \
  "hack/*kubeconfig.yaml" \
  "hack/kind*.yaml" \
  "hack/values*.yaml"

rm -f \
  "config/pki/*.json" \
  "config/pki/*-key.pem" \
  "config/pki/*.pem"

rm -f "config/cert.yaml"
```

---

## Troubleshooting

- **Cluster Creation Issues**: Ensure Docker is running and `kind` is installed correctly.
- **KCM Deployment Issues**: Check Helm logs for errors during the KCM installation.

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
