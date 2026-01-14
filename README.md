# K0rdent Demo

This repository demonstrates how to set up a Kubernetes cluster using `kind`, deploy KCM (Kubernetes Cluster Manager), and manage workloads.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
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

---

## Setup Instructions

Use kind to create the management cluster, orbstack or docker-desktop can also be used.

### 1. Generate Kind Config File

Run the following command to generate a `kind` configuration file:

```shell
stat hack/kind-kcm.yaml >/dev/null 2>&1 || cat <<EOF >hack/kind-kcm.yaml
---
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
name: kind
networking:
  apiServerAddress: 127.0.0.1
  apiServerPort: 6443
featureGates:
  UserNamespacesSupport: true
nodes:
  - role: control-plane
    extraMounts:
      - # We are mounting the Docker socket to allow KCM to manage containers
        # on the host system (for DockerCluster).
        hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
        listenAddress: 0.0.0.0
      - containerPort: 443
        hostPort: 443
        protocol: TCP
        listenAddress: 0.0.0.0
      - # Port for KCM management
        containerPort: 30443
        hostPort: 30443
        protocol: TCP
        listenAddress: 0.0.0.0
EOF
```

### 2. Create Management Cluster

Create a management cluster using the generated configuration:

```shell
grep -qa kind-kcm <(kind get clusters) >/dev/null 2>&1 || kind create cluster \
  --config hack/kind-kcm.yaml \
  --name kind-kcm
```

### 3. Install KCM

Deploy KCM using Helm:

```shell
kcm_templates_url=oci://ghcr.io/labsonline/charts/kcm
kcm_version=1.3.5

# install gwapi (optional)
helm upgrade gwapi $kcm_templates_url \
  --rollback-on-failure --install \
  --namespace kube-system \
  --version 0.1.0 \
  --values <(
    cat <<EOF
gateway-helm:
  config:
    envoyGateway:
      gateway:
        controllerName: gateway.envoyproxy.io/gatewayclass-controller
      provider:
        type: Kubernetes
      logging:
        level:
          default: info
  service:
    type: LoadBalancer
    annotations:
      external-dns.alpha.kubernetes.io/hostname: local
  topologyInjector:
    enabled: true
EOF
)

# create kcm namespace
kubectl create namespace kcm-system --dry-run=client -o yaml | kubectl apply -f -

# create NEXTAUTH_SECRET
NEXTAUTH_SECRET=$(openssl rand -hex 8)
kubectl create secret generic k0rdent-nextauth \
  --namespace kcm-system \
  --from-literal=NEXTAUTH_SECRET=$NEXTAUTH_SECRET \
  --dry-run=client -o yaml | kubectl apply -f -

# create values file for kcm
cat <<eof >/tmp/values.kcm.yaml
controller: &controller
  templatesRepoURL: oci://registry.mirantis.com/k0rdent-enterprise/charts # Enterprise
  # templatesRepoURL: oci://ghcr.io/k0rdent/kcm/charts                      # OSS
global: &global
  controller:
    <<: *controller
  # regional:
  #   cert-manager:
  #     config:
  #       enableGatewayAPI: true
kcm:
  <<: *global
  install: false
k0rdent-enterprise:
  <<: *global
  install: true
  k0rdent-ui:
    enabled: true
eof

# deploy kcm
helm upgrade kcm $kcm_templates_url \
  --install --rollback-on-failure --create-namespace \
  --namespace kcm-system \
  --values /tmp/values.kcm.yaml \
  --version $kcm_version \
  --wait

# create values file for kcm resource
cat <<eof >/tmp/values.kcmres.yaml
controller:
  templatesRepoURL: oci://registry.mirantis.com/k0rdent-enterprise/charts # Enterprise
  # templatesRepoURL: oci://ghcr.io/k0rdent/kcm/charts                      # OSS
version: 1.2.2  # Enterprise
# version: 1.6.0  # OSS
templates:
  # Enterprise
  capi: cluster-api-1-0-7
  kcm: k0rdent-enterprise-1-2-2
  regional: kcm-regional-1-2-2
  # OSS
  # capi: cluster-api-1-0-9
  # kcm: kcm-1-6-0
  # regional: kcm-regional-1-6-0
management:
  enabled: true
  access:
    enabled: true
    rules:
      - targetNamespaces:
          list:
            - default
        credentials:
          - docker-cluster-cred
          - remote-cluster-cred
        clusterTemplateChains:
          - adopted-cluster
          - docker-hosted-cp
          - remote-cluster
        serviceTemplateChains:
          - core
          - addon
  providers:
    - name: cluster-api-provider-docker
    - name: cluster-api-provider-k0sproject-k0smotron
    - name: projectsveltos
release:
  enabled: true
  providers:
    - name: projectsveltos
      template: projectsveltos-1-1-1
    # Enterprise
    - name: cluster-api-provider-docker
      template: cluster-api-provider-docker-1-0-5
    - name: cluster-api-provider-k0sproject-k0smotron
      template: cluster-api-provider-k0sproject-k0smotron-1-0-12
    # OSS
    # - name: cluster-api-provider-docker
    #   template: cluster-api-provider-docker-1-0-7
    # - name: cluster-api-provider-k0sproject-k0smotron
    #   template: cluster-api-provider-k0sproject-k0smotron-1-0-14
    # - name: cluster-api-provider-docker
    #   template: cluster-api-provider-docker-1-0-7
eof

# deploy kcm resource
helm upgrade kcm-resource $kcm_templates_url \
  --install --rollback-on-failure --create-namespace \
  --namespace kcm-system \
  --values /tmp/values.kcmres.yaml \
  --version $kcm_version \
  --wait

# wait for KCM to be ready
kubectl wait \
  --for condition=Ready \
  --namespace kcm-system \
  --timeout 720s \
  Management/kcm
```

### 4. Deploy Management Workload

Deploy your management workload using `kustomize`:

```shell
PRIVATE_SSH_KEY_B64=$(cat ~/.ssh/id_ed25519 | base64 -w0)
PRIVATE_SSH_KEY_B64="${PRIVATE_SSH_KEY_B64}" envsubst < template/cluster/remote/example.env >template/cluster/remote/.env

kubectl apply -f <(kustomize build)
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
# generate kind config
stat hack/kind.yaml >/dev/null 2>&1 || cat <<EOF >hack/kind.yaml
---
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
name: kind
networking:
  # disableDefaultCNI: true
  apiServerAddress: 0.0.0.0
  apiServerPort: 6443
featureGates:
  UserNamespacesSupport: true
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
        listenAddress: 0.0.0.0
      - containerPort: 443
        hostPort: 443
        protocol: TCP
        listenAddress: 0.0.0.0
EOF

# create kind cluster
grep -qa kind <(kind get clusters) >/dev/null 2>&1 || kind create cluster --config hack/kind.yaml

# create credential
BASE64_KUBECONFIG=$(kind get kubeconfig --name="kind" | base64 -w0)
BASE64_KUBECONFIG="${BASE64_KUBECONFIG}"  envsubst < deployment/adopted/example.env >deployment/adopted/.env
```

Create a workload cluster:

```shell
kustomize build ./deployment | kubectl apply -f -

# Wait for kubeconfig to be created
echo "Waiting for kubeconfig to be created..."
while true; do
  secret_name=$(kubectl get secret -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep kubeconfig | head -n1)
  if [[ -n "${secret_name}" ]]; then
    echo "Secret ${secret_name} found."
    break
  fi
  echo "Secret not found. Retrying in 5 seconds..."
  sleep 5
done

# Export the kubeconfig from the secret to hack/dev-docker-kubeconfig.yaml
kubectl get secret dev-docker-kubeconfig -o jsonpath='{.data.value}' | base64 --decode > hack/dev-docker-kubeconfig.yaml
sed -i '' 's/server: https:\/\/.*:30443/server: https:\/\/127.0.0.1:30443/' hack/dev-docker-kubeconfig.yaml
echo "Exported kubeconfig to hack/dev-docker-kubeconfig.yaml"
```

## Cleanup

To clean up the management cluster and resources, run:

```shell
kubectl delete -f <(kustomize build ./deployment) || true
kind delete cluster --name kind
kind delete cluster --name kind-kcm
rm -f hack/kind-kcm.yaml /tmp/values.yaml hack/dev-docker-kubeconfig.yaml hack/kind.yaml
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
