# Deploying the 3 tier application on K3 cluster hosted on Raspberry Pi 4
```
kubectl apply -f 1-database.yaml
kubectl apply -f 2-backend.yaml
kubectl apply -f 3-frontend.yaml
kubectl apply -f 4-ingress.yaml
```

## Exposing the App: Ingress Configuration
K3s ships natively with the Traefik Ingress Controller out of the box.
You do not need to install an external NGINX setup. Create an Ingress rule to route external traffic hitting your Raspberry Pi’s IP straight into your frontend service.

## MetalLB (Layer 2 Network Load Balancing)
To take your Raspberry Pi 4 K3s cluster to a production-ready level, you can replace K3s's basic default tools with MetalLB for network load balancing and Longhorn for distributed, replicated block storage.
Before starting, disable K3s's built-in tools to prevent conflicts.
Traefik can stay, but K3s's default servicelb (Klipper) must be disabled so MetalLB can manage IP addresses.
Edit your K3s service file or restart configuration to include the --disable servicelb flag.

By default, MetalLB provides a static external IP address to your service for as long as that service exists in your cluster.
If a node crashes, a pod restarts, or the entire K3s cluster reboots, MetalLB will reassign the exact same IP address to that service.

## Longhorn (Distributed High-Availability Storage)
Longhorn turns the local storage of your individual Raspberry Pi nodes into a single, replicated, resilient storage pool.
If one Pi fails, your database or stateful workloads can instantly spin up on another Pi without losing data.

### Prerequisite: Install open-iscsi (You must run this command natively on every Raspberry Pi node)
```
sudo apt-get update && sudo apt-get install -y open-iscsi util-linux jq
```

## Install MetalLB + Longhorn via Helm

### Add the MetalLB Helm repository
helm repo add metallb https://metallb.github.io/metallb
helm repo update

### Create the namespace and deploys MetalLB components
helm install metallb metallb/metallb --namespace metallb-system --create-namespace

### Install with custom values
helm install metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace \
  -f metallb-values.yaml

## Deploy MetalLB
```
kubectl apply -f metallb-config.yaml

Check if IP pools are configured:
kubectl get ipaddresspools -n metallb-system

View detailed speaker logs:
kubectl logs -n metallb-system -l app=metallb,component=speaker --tail=100
```

### K3s ServiceLB Conflict
If you installed K3s with the default ServiceLB (Klipper), configure all K3s server nodes to disable it and restart K3s:
```
disable:
- servicelb
```

After K3s is running with ServiceLB disabled, remove any old ServiceLB DaemonSets
```
kubectl get daemonset -n kube-system -o name | grep '^daemonset.apps/svclb-' | xargs -r kubectl delete -n kube-system
```
Or reinstall K3s with ServiceLB disabled
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=servicelb" sh -
```

### Add the Longhorn Helm repository
helm repo add longhorn https://longhorn.io
helm repo update

### Create the namespace and install Longhorn
kubectl create namespace longhorn-system
helm install longhorn longhorn/longhorn --namespace longhorn-system

### Update Your 3-Tier App to Use Longhorn
Longhorn automatically registers itself as a Kubernetes StorageClass named longhorn.
To migrate your database from the fragile local node storage to replicated storage, change your database PVC definition (1-database.yaml)

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn # Tells K3s to use Longhorn instead of local-path
  resources:
    requests:
      storage: 5Gi # Longhorn easily handles volume scaling
```

### Deploying the Longhorn Dashboard
```
kubectl apply -f longhorn-ui.yaml
```
Run kubectl get svc -n longhorn-system to see the External IP address that MetalLB assigned to your dashboard. Type that IP into your browser to manage your replicated cluster storage.