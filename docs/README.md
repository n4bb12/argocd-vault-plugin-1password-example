# Argo CD Vault Plugin Example

## Install Command-Line Tools

- Install `go` — https://go.dev/doc/install
- Install `kubectl` — https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
- Install `kind` — https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries
- Install `argocd` — https://github.com/argoproj/argo-cd/releases/latest
- Install `helm` — https://helm.sh/docs/intro/install/
- Install `op` — https://developer.1password.com/docs/cli/get-started#install

## Create Cluster

```bash
kind create cluster --name argocd
```

## Create 1Password Integration

- https://developer.1password.com/docs/connect/get-started/#step-1-set-up-a-secrets-automation-workflow
- Create a new vault named `argocd`
- Create a new "Secure Note" named `example-app` in the `argocd` vault with an entry `EXAMPLE_KEY=SECRET_VALUE`
- Create a new integration for the vault and save the credentials file to `secrets/1password-credentials.json`
- Create a new integration access token for the vault and save it to `secrets/1password-token.txt`

## Install Argo CD

- https://argocd-vault-plugin.readthedocs.io/en/stable/config/#kubernetes-secret
- https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#kustomize

```
kubectl create ns argocd

kubectl -n argocd create secret generic argocd-vault-plugin \
  --from-literal=AVP_TYPE=1passwordconnect \
  --from-literal=OP_CONNECT_HOST=http://onepassword-connect:8080 \
  --from-literal=OP_CONNECT_TOKEN=$(cat secrets/1password-token.txt)

kubectl apply -k argocd-vault-plugin
kubectl -n argocd get pod
```

## Deploy 1Password Connect Server

- https://developer.1password.com/docs/connect/get-started/#step-2-deploy-1password-connect-server

```bash
helm repo add 1password https://1password.github.io/connect-helm-charts/

helm install 1password-connect 1password/connect \
  --namespace argocd \
  --set-file connect.credentials=secrets/1password-credentials.json
```

## Troubleshooting

[TROUBLESHOOTING.md](TROUBLESHOOTING.md)
