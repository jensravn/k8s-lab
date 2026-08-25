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

