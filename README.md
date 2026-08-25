# k8s-lab

A personal lab for learning Kubernetes alongside the CKA course.

## Cluster

Cheap by three choices: Free tier control plane, two small burstable nodes,
and a 32 GB OS disk instead of the 128 GB default.

```bash
az group create --name k8s-lab-rg --location northeurope

az aks create --resource-group k8s-lab-rg --name k8s-lab \
  --tier free --node-count 2 \
  --node-vm-size Standard_B2als_v2 --node-osdisk-size 32 \
  --generate-ssh-keys

az aks get-credentials --resource-group k8s-lab-rg --name k8s-lab
```

Stopping the cluster deallocates the nodes so compute stops billing, and it comes back in a couple of minutes.

```bash
az aks stop  --resource-group k8s-lab-rg --name k8s-lab
az aks start --resource-group k8s-lab-rg --name k8s-lab
```

To get rid of it entirely: `az group delete --name k8s-lab-rg --yes`

## Web app

React + Vite, served by nginx.

```bash
cd apps/web
pnpm install
pnpm dev            # http://localhost:5173
```

The cluster nodes are amd64, so the image must be too. `--platform` is
only needed when building on a different architecture, such as an ARM
laptop; get it wrong and the pod fails with `exec format error`.

```bash
docker build --platform linux/amd64 --tag k8s-lab-web:dev apps/web
docker run --rm --publish 8080:8080 k8s-lab-web:dev   # http://localhost:8080
```

Push to GitHub Container Registry. The token needs `write:packages`:

```bash
gh auth refresh --hostname github.com --scopes write:packages,read:packages
```

```bash
gh auth token | docker login ghcr.io \
  --username "$(gh api user --jq .login)" --password-stdin

docker tag k8s-lab-web:dev ghcr.io/jensravn/k8s-lab/web:0.1.0
docker push ghcr.io/jensravn/k8s-lab/web:0.1.0
```

A new GHCR package is private by default. Either make it public under the
package settings on GitHub, or give the cluster an image pull secret.

These steps are manual on purpose while the shape of the app is still
changing. They move to GitHub Actions once it settles — build on push,
tag with the commit SHA, and deploy. Avoid `:latest`: Kubernetes cannot tell
that a tag it already has has changed, so a rollout does nothing.

## Deploy

The GHCR package is private, so the cluster needs credentials to pull it.
Use a token with only `read:packages` — the token `gh` holds also carries
`repo`, and a leaked secret should not hand over every repository.

```bash
kubectl create secret docker-registry ghcr \
  --docker-server=ghcr.io \
  --docker-username="$(gh api user --jq .login)" \
  --docker-password=<personal access token>
```

The secret is namespaced: it has to live in the same namespace as the pods
that reference it. Its contents are base64, which is encoding, not
encryption — anyone who can read the secret can read the token.

```bash
kubectl apply -f manifests/web/
kubectl get pods --output wide
kubectl get service web --watch      # EXTERNAL-IP stays <pending> for a minute
```

The Service is `ClusterIP`: only reachable inside the cluster. Nothing
outside reaches it yet — that is the next section's job.

The two replicas are spread across nodes, so draining one shows Kubernetes
rescheduling while the site keeps answering:

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl get pods --output wide
kubectl uncordon <node>
```

