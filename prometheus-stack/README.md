# Prometheus stack

This wrapper chart installs the official `kube-prometheus-stack` chart as a dependency. The release includes Prometheus, Alertmanager, Grafana, and the Prometheus Operator. The Grafana HTTPS Ingress is maintained locally in `templates/ingress.yaml`.

## Prerequisites

- Install `../storage` first.
- Install an NGINX Ingress Controller.
- Confirm that the required StorageClasses exist:

```sh
kubectl get storageclass local-prometheus local-grafana local-alertmanager
```

Create a private values file for the Grafana administrator password and ingress TLS certificate:

```sh
cp prometheus-stack/private-values.example.yaml prometheus-stack/private-values.yaml
# Replace kube-prometheus-stack.grafana.adminPassword,
# ingress.tls.certificate, and ingress.tls.privateKey.
```

`prometheus-stack/private-values.yaml` is ignored by Git.

Grafana is exposed at `https://grafana.ibek.dev`. The certificate in `prometheus-stack/private-values.yaml` must cover `grafana.ibek.dev`.

## Install

From the repository root:

```sh
helm dependency build ./prometheus-stack
cp prometheus-stack/private-values.example.yaml prometheus-stack/private-values.yaml
# Replace kube-prometheus-stack.grafana.adminPassword,
# ingress.tls.certificate, and ingress.tls.privateKey.

helm upgrade --install prometheus-stack ./prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-stack/private-values.yaml
```

## Verify

```sh
helm status prometheus-stack --namespace monitoring
kubectl get pods,pvc,service --namespace monitoring
kubectl get ingress --namespace monitoring
```

Point the `grafana.ibek.dev` DNS record at the external address of the ingress controller, then verify:

```sh
curl --fail --show-error https://grafana.ibek.dev/api/health
```

## Upgrade

```sh
helm dependency update ./prometheus-stack
helm upgrade prometheus-stack ./prometheus-stack \
  --namespace monitoring \
  -f prometheus-stack/private-values.yaml
```
