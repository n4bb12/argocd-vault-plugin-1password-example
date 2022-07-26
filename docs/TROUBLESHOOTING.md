## Troubleshooting

### Access Argo CD UI

- https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli
- https://argo-cd.readthedocs.io/en/stable/getting_started/#3-access-the-argo-cd-api-server

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl -n argocd port-forward svc/argocd-server 9999:443
```

### Configure Argo CD CLI

```bash
kubectl config set-context --current --namespace=argocd
argocd login --core
```

### Create a Test Application

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app sync guestbook

kubectl -n default port-forward svc/guestbook-ui 8888:80
```

### Create a Secret Without 1Password

```bash
argocd app create example-without-op \
  --repo https://github.com/n4bb12/argocd-vault-plugin-example.git \
  --path example-without-op \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace example

argocd app sync example-app-without-op

kubectl -n example get secret
```

### Create a Secret With 1Password

```bash
argocd app create example \
  --repo https://github.com/n4bb12/argocd-vault-plugin-example.git \
  --path example \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace example

argocd app sync example-app

kubectl -n example get secret
```

### Access 1Password Connect Server

#### By Proxying the Kubernetes Service

```bash
kubectl -n argocd port-forward svc/onepassword-connect 7777:8080
```

#### By Starting a Separate 1Password Connect Server as Docker Container

```bash
docker-compose up -d
```

#### And Make a Request

```bash
export OP_API_TOKEN=$(cat secrets/1password-token.txt)

curl \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $OP_API_TOKEN" \
  http://localhost:7777/v1/vaults
```
