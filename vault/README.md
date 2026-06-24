# Vault

This wrapper chart installs the official HashiCorp Vault chart as a dependency. Vault runs as a three-node Raft cluster and consumes the `local-vault` StorageClass. The HTTPS Ingress is maintained locally in `templates/ingress.yaml`.

## Prerequisites

- Install `../storage` first.
- Install an NGINX Ingress Controller.
- Confirm that the StorageClass exists:

```sh
kubectl get storageclass local-vault
```

Vault is exposed at `https://vault.k8s.ibek.dev`. TLS terminates at the ingress, while the Vault listener uses HTTP inside the cluster.

Create a private values file for the ingress TLS certificate:

```sh
cp vault/private-values.example.yaml vault/private-values.yaml
# Replace ingress.tls.certificate and ingress.tls.privateKey in vault/private-values.yaml.
```

`vault/private-values.yaml` is ignored by Git. The certificate must cover `vault.k8s.ibek.dev`.

## Install

From the repository root:

```sh
helm dependency build ./vault
cp vault/private-values.example.yaml vault/private-values.yaml
# Replace ingress.tls.certificate and ingress.tls.privateKey in vault/private-values.yaml.

helm upgrade --install vault ./vault \
  --namespace vault \
  --create-namespace \
  -f vault/private-values.yaml
```

## Verify

```sh
helm status vault --namespace vault
kubectl get pods,pvc,service --namespace vault
kubectl get ingress --namespace vault
kubectl exec --namespace vault vault-0 -- vault status
```

Point the `vault.k8s.ibek.dev` DNS record at the external address of the ingress controller. After Vault is initialized and unsealed, verify:

```sh
curl --fail --show-error https://vault.k8s.ibek.dev/v1/sys/health
```

A new Vault cluster starts sealed and must be initialized and unsealed according to your key-management procedure. Store unseal keys and the initial root token outside this repository.

## Upgrade

```sh
helm dependency update ./vault
helm upgrade vault ./vault \
  --namespace vault \
  -f vault/private-values.yaml
```
