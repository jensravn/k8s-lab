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

The Service is `ClusterIP`: only reachable inside the cluster. Everything
from outside arrives through the Gateway below. The HTTPRoute applied here
sits idle until that Gateway exists, and attaches on its own once it does.

The two replicas are spread across nodes, so draining one shows Kubernetes
rescheduling while the site keeps answering:

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl get pods --output wide
kubectl uncordon <node>
```

## Traffic and DNS

Everything from outside arrives through one shared entry point:

```
jravn.com            A record in Google Cloud DNS
      |
$INGRESS_IP          static public IP in Azure
      |
Gateway              Envoy Gateway, owns the listener
      |
HTTPRoute            jravn.com -> web
      |
web Service          ClusterIP, internal only
      |
2 pods               one per node
```

Gateway API splits what `Ingress` crammed into a single resource. The
`Gateway` owns the listener and the only public IP; an `HTTPRoute` attaches
to it and says where one hostname's traffic goes. A second app is another
HTTPRoute — no extra load balancer, no extra IP, no DNS work beyond the one
record.

The split is also a split of ownership. Whoever runs the cluster owns the
GatewayClass and the Gateway; whoever ships an app owns its route, and can
do so without touching shared infrastructure.

### Static IP

An Azure load balancer gets a fresh IP every time it is recreated, which is
useless to point DNS at. Create the address up front instead, in the node
resource group AKS manages:

```bash
NODE_RG=$(az aks show --resource-group k8s-lab-rg --name k8s-lab \
  --query nodeResourceGroup --output tsv)

INGRESS_IP=$(az network public-ip create --resource-group "$NODE_RG" \
  --name k8s-lab-ingress-ip --sku Standard --allocation-method Static \
  --query publicIp.ipAddress --output tsv)
```

Keep `$INGRESS_IP` — the DNS record below needs it. A rebuilt cluster gets a
different address, so nothing downstream may hardcode one.

### Gateway controller

```bash
helm upgrade --install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version 1.9.0 \
  --namespace envoy-gateway-system --create-namespace
```

The chart brings the Gateway API CRDs and the controller, and nothing else.
It does not create a GatewayClass — that is the cluster's decision, and it
lives in `manifests/gateway/`.

### Gateway and routes

```bash
kubectl apply -f manifests/gateway/
kubectl get gateway web --watch      # ADDRESS stays empty for a minute
```

Envoy Gateway creates the LoadBalancer Service itself, one per Gateway, so
there is no Service of your own to annotate. The way in is the GatewayClass:
it points at an `EnvoyProxy` resource holding the infrastructure settings for
every Gateway in the class — here, the annotation that claims
`k8s-lab-ingress-ip` instead of accepting a fresh address.

One listener, on port 80. The HTTPRoute applied with the app attaches to it
and sends `jravn.com` to the `web` Service.

### DNS

The domain is registered on Google Cloud, so the record lives there. The
cluster being on Azure makes no difference; DNS is independent of both.

```bash
gcloud dns record-sets update jravn.com. \
  --project=jensravn --zone=jravn-com \
  --type=A --ttl=300 --rrdatas="$INGRESS_IP"
```

`update`, not `create` — the zone outlives the cluster, so on a rebuild the
record is already there and `create` fails. Use `create` only for a name the
zone has never held.

The site lives at the apex rather than a subdomain, which forces an `A`
record: a zone apex cannot be a `CNAME`. That is the whole reason the address
is created up front and static.

