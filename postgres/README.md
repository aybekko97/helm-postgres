# Postgres

This Helm chart creates a CloudNativePG PostgreSQL cluster, a TLS Secret, and a NodePort service for the primary instance. It consumes the `local-postgres` StorageClass created by the storage component.

## Prerequisites

- Install `../storage` first.
- Install the CloudNativePG operator in the cluster.
- Confirm that `local-postgres` exists:

```sh
kubectl get storageclass local-postgres
```

## Configure TLS

Create the ignored private override file:

```sh
cp postgres/private-values.example.yaml postgres/private-values.yaml
```

Fill in `certificate`, `ca_certificate`, and `privateKey` with base64-encoded values. Never commit `private-values.yaml`.

### Why the secret file is separate

YAML has no standard file-import directive, and Helm does not evaluate templates or imports inside `values.yaml`. A chart also cannot silently read an arbitrary file from the machine running Helm.

Helm instead merges values in layers:

1. `postgres/values.yaml` is loaded automatically as the chart default.
2. `-f postgres/private-values.yaml` is loaded afterward.
3. Matching values from the private file override the defaults.

For example, the chart safely commits an empty default:

```yaml
tls:
  privateKey: ""
```

The ignored override supplies the real value:

```yaml
tls:
  privateKey: "BASE64_ENCODED_PRIVATE_KEY"
```

You do not need to pass the default file explicitly. These commands are equivalent:

```sh
helm upgrade --install postgres-local ./postgres \
  -f postgres/private-values.yaml

helm upgrade --install postgres-local ./postgres \
  -f postgres/values.yaml \
  -f postgres/private-values.yaml
```

When multiple `-f` options are provided, the rightmost file has the highest priority.

## Install

From the repository root:

```sh
helm lint ./postgres -f postgres/private-values.yaml

helm upgrade --install postgres-local ./postgres \
  --namespace postgres \
  --create-namespace \
  -f postgres/private-values.yaml
```

## Verify

```sh
helm status postgres-local --namespace postgres
kubectl get clusters.postgresql.cnpg.io --namespace postgres
kubectl get pods,pvc,service --namespace postgres
```

Wait until all database instances are ready and their PVCs are bound.

## Upgrade

```sh
helm upgrade postgres-local ./postgres \
  --namespace postgres \
  -f postgres/private-values.yaml
```

Rotate the TLS private key if it has ever been committed or shared. Adding a file to `.gitignore` does not remove it from Git history.

## Production secret management

The current chart renders the TLS Secret, which means the rendered secret is stored as part of Helm's release data in Kubernetes. For production, prefer an externally managed Secret using a system such as External Secrets Operator, SOPS, or Sealed Secrets, and configure the Postgres chart to reference that existing Secret instead of rendering private key material through Helm.
