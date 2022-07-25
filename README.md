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
kind get clusters
kubectl cluster-info --context kind-argocd
```

## Install Argo CD with Vault Plugin

- https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#kustomize

```bash
kubectl apply -k argocd-vault-plugin
```

## Access Argo CD UI

- https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli
- https://argo-cd.readthedocs.io/en/stable/getting_started/#3-access-the-argo-cd-api-server

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl -n argocd port-forward svc/argocd-server 9999:443
```

## Register External Cluster (optional)

- https://argo-cd.readthedocs.io/en/stable/getting_started/#5-register-a-cluster-to-deploy-apps-to-optional

```bash
kubectl config get-contexts -o name

argocd cluster add kind-argocd
```

## Configure 1Password CLI

- https://developer.1password.com/docs/cli/get-started

```bash
eval $(op account add --signin)

export OP_SESSION_xxxxxx=xxxxxx

op connect server create argocd
```

## Create 1Password Integration

- https://developer.1password.com/docs/connect/get-started/#step-1-set-up-a-secrets-automation-workflow
- Create a new vault
- Create a new secret note in the vault
  - type: Secure Note
  - name: example-app
  - ENV_NAME: SECRET_VALUE
- Create a new integration for the vault and save the credentials file to `secrets/1password-credentials.json`
- Create a new integration access token for the vault and save it to `secrets/1password-token.txt`

```bash
op connect server create argocd
op connect token create argocd --server argocd --vaults argocd
```

## Deploy 1Password Connect Server (Helm)

- https://developer.1password.com/docs/connect/get-started/#step-2-deploy-1password-connect-server

```bash
kubectl -n argocd create secret generic argocd-vault-plugin \
  --from-literal=AVP_TYPE=1passwordconnect \
  --from-literal=OP_CONNECT_HOST=http://onepassword-connect:8080 \
  --from-literal=OP_CONNECT_TOKEN=$(cat secrets/1password-token.txt)

helm repo add 1password https://1password.github.io/connect-helm-charts/

helm delete 1password-connect || true

# only connect
helm install 1password-connect 1password/connect \
  --namespace argocd \
  --set-file connect.credentials=secrets/1password-credentials.json

# connect and operator
helm install 1password-connect 1password/connect \
  --namespace argocd \
  --set-file connect.credentials=secrets/1password-credentials.json \
  --set operator.create=true \
  --set operator.token.value=$(openssl rand -base64 32)
```

## Deploy 1Password Connect Server (Docker)

```bash
docker-compose up -d

export OP_API_TOKEN=$(cat secrets/1password-token.txt)

curl \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $OP_API_TOKEN" \
  http://localhost:7777/v1/vaults
```

## Configure Argo CD CLI

```bash
kubectl config set-context --current --namespace=argocd
argocd login --core
```

## Create A Test Application

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app get guestbook
argocd app sync guestbook

kubectl -n default port-forward svc/guestbook-ui 8888:80
```

## Test Vault Plugin

```bash
argocd repo add https://github.com/n4bb12/argocd-vault-plugin-example.git \
  --username n4bb12 \
  --password $(cat secrets/github-token.txt)

kubectl create ns example

argocd app create example-app-without-op \
  --repo https://github.com/n4bb12/argocd-vault-plugin-example.git \
  --path example-app-without-op \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace example

argocd app create example-app \
  --repo https://github.com/n4bb12/argocd-vault-plugin-example.git \
  --path example-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace example
```
