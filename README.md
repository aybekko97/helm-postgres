# Local Kubernetes platform

This repository contains four independently deployable components. Install them in the order below because the application charts depend on StorageClasses created by the storage chart.

## Prerequisites

- A working Kubernetes cluster and `kubectl` context
- Helm 3
- CloudNativePG operator installed before Postgres
- NGINX Ingress Controller installed before Vault and Prometheus stack
- Local storage directories created on every node listed in `storage/values.yaml`

## Installation order

### 1. Local storage

Creates workload-specific StorageClasses and local PersistentVolumes.

```sh
helm upgrade --install local-storage ./storage
```

Verify that the storage resources exist before continuing:

```sh
kubectl rollout status daemonset/local-storage-prepare-directories
kubectl get storageclass
kubectl get persistentvolume
```

See [storage/README.md](storage/README.md) for node preparation and configuration.

### 2. Postgres

Creates the CloudNativePG cluster, TLS Secret, and external NodePort service.

```sh
cp postgres/private-values.example.yaml postgres/private-values.yaml
# Fill postgres/private-values.yaml with base64-encoded TLS values.

helm upgrade --install postgres-local ./postgres \
  --namespace postgres \
  --create-namespace \
  -f postgres/private-values.yaml
```

See [postgres/README.md](postgres/README.md) for prerequisites and verification.

### 3. Vault

Installs the local Vault wrapper chart and exposes it at `https://vault.k8s.ibek.dev`.

```sh
helm dependency build ./vault
cp vault/private-values.example.yaml vault/private-values.yaml
# Replace ingress.tls.certificate and ingress.tls.privateKey in vault/private-values.yaml.

helm upgrade --install vault ./vault \
  --namespace vault \
  --create-namespace \
  -f vault/private-values.yaml
```

See [vault/README.md](vault/README.md) for initialization and status checks.

### 4. Prometheus stack

Installs the local Prometheus wrapper chart, including Prometheus, Alertmanager, Grafana, and the Prometheus Operator. Grafana is exposed at `https://grafana.ibek.dev`.

```sh
helm dependency build ./prometheus-stack
cp prometheus-stack/private-values.example.yaml prometheus-stack/private-values.yaml
# Replace the Grafana admin password and ingress TLS values in prometheus-stack/private-values.yaml.

helm upgrade --install prometheus-stack ./prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-stack/private-values.yaml
```

See [prometheus-stack/README.md](prometheus-stack/README.md) for access and verification.

## Validate local charts

```sh
helm lint ./storage
helm template local-storage ./storage >/dev/null

helm lint ./postgres -f postgres/private-values.yaml
helm template postgres-local ./postgres \
  --namespace postgres \
  -f postgres/private-values.yaml >/dev/null

helm dependency build ./vault
helm lint ./vault -f vault/private-values.yaml
helm template vault ./vault \
  --namespace vault \
  -f vault/private-values.yaml >/dev/null

helm dependency build ./prometheus-stack
helm lint ./prometheus-stack -f prometheus-stack/private-values.yaml
helm template prometheus-stack ./prometheus-stack \
  --namespace monitoring \
  -f prometheus-stack/private-values.yaml >/dev/null
```

Do not remove the storage release while application PVCs still use its volumes. Local PVs use the `Retain` reclaim policy, so uninstalling an application does not delete its data from the nodes.
