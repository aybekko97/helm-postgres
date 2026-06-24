# Local storage

This local Helm chart creates one StorageClass per workload and one local PersistentVolume per configured node and storage pool.

## Prerequisites

Review `values.yaml`, particularly the Kubernetes node names and host paths:

```sh
kubectl get nodes
```

Every configured host path must exist on every listed node before an application pod can mount its local PV. This chart includes a `prepareDirectories` DaemonSet that creates those directories automatically on the configured nodes.

Keep these paths unique per workload. Reusing the same path for Postgres and Grafana, for example, lets the containers rewrite each other's ownership and permissions.

Default workload paths:

| Workload | StorageClass | Host path |
| --- | --- | --- |
| Postgres | `local-postgres` | `/mnt/disks/postgres-data` |
| Prometheus | `local-prometheus` | `/mnt/disks/prometheus-data` |
| Grafana | `local-grafana` | `/mnt/disks/grafana-data` |
| Alertmanager | `local-alertmanager` | `/mnt/disks/alertmanager-data` |
| Vault | `local-vault` | `/mnt/disks/vault-data` |

## Install

From the repository root:

```sh
helm lint ./storage
helm upgrade --install local-storage ./storage
```

StorageClasses and PersistentVolumes are cluster-scoped, so no namespace is required.

## Verify

```sh
helm status local-storage
kubectl get daemonset local-storage-prepare-directories
kubectl get pods -l app.kubernetes.io/name=local-storage-prepare-directories
kubectl get storageclass
kubectl get persistentvolume
```

The chart should create five StorageClasses and fifteen PVs with the default three-node configuration.

The directory-preparation DaemonSet mounts the host root at `/host`, creates every configured `hostPath`, and applies `prepareDirectories.mode` from `values.yaml`. The default mode is `0777`, which is convenient for a local lab because different workloads run as different Linux users. Tighten it if you standardize workload `runAsUser`, `runAsGroup`, and `fsGroup`.

## Upgrade

Edit `storage/values.yaml`, render the proposed resources, and then upgrade:

```sh
helm template local-storage ./storage
helm upgrade local-storage ./storage
```

Changing PV names, node names, StorageClass names, or host paths may disrupt existing claims. Do not uninstall this release while application PVCs still depend on it.

If a PVC has already bound to the wrong StorageClass or host path, updating Helm values is not enough; Kubernetes will keep the existing PVC binding. Back up any data you need, delete the wrong workload PVC, and reinstall or upgrade the workload so it creates a new PVC against the corrected StorageClass.
