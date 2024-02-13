# Development Environment Setup
## k3s

Lightweight Kubernetes. Production ready, easy to install, half the memory, all in a binary less than 100 MB.

Assumptions:
- Host OS: Linux
- Root access

```bash
curl -sfL https://get.k3s.io | sh -

# Obtain k3s Credentials
sudo cp /etc/rancher/k3s/k3s.yaml  ~/.kube/config
sudo chown $USER:$USER ~/.kube/config

# Post installation verification
sudo kubectl get nodes 
```
## argocd

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

Assumptions:
- Running Kubernetes cluster

```bash
# Setting environment variable
export NAMESPACE=argocd

# Create kubernetes namespace for Argo CD
kubectl create ns $NAMESPACE

#Argo CD Server Installation
kubectl -n $NAMESPACE apply  -f https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml

# Post installation verification
kubectl -n $NAMESPACE get pods #all pods must be running

# Getting credentials
kubectl -n argocd get secret argocd-initial-admin-secret -o yaml | grep -o 'password: .*' | sed -e s"/password\: //g" | base64 -d

# Proxy the argocd-server
kubectl -n $NAMESPACE port-forward --address 0.0.0.0  svc/argocd-server -n argocd 8080:443

# Argo CD Client Installation
# https://argo-cd.readthedocs.io/en/stable/cli_installation/
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Login to argocd
argocd login localhost:8080

# Cluster Boostrapping Official Documentation
# https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/

# Self manage the argocd app
argocd app create fleet-infra \
    --dest-namespace argocd \
    --dest-server https://kubernetes.default.svc \
    --repo https://github.com/mlangeles/fleet-infra \
    --path apps  \
    --label app=fleet-infra
argocd app sync fleet-infra
```

## helm

Package manager for kubernetes

Assumptions:
- Running Kubernetes cluster

```bash
# Source 
# https://helm.sh/docs/intro/

# Helm Script Installation
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm ./get_helm.sh
```

### Test Helm
```bash
export NGINX=nginx
# Add NGINX Helm Repository
helm repo add nginx https://kubernetes.github.io/ingress-nginx

# NGINX Helm Chart Installation
kubectl create ns $NGINX
helm -n $NGINX install $NGINX nginx/ingress-nginx --version 4.9.1

# NGINX Store Installation Template to File 
helm template $NGINX nginx/ingress-nginx --version 4.9.1 > test.yaml

# NGINX Helm Chart Uninstallion from Namespace $NGINX
helm -n $NGINX uninstall $NGINX
```

# TODO
- [x] Network Controller 
- [ ] Storage Controller
- [ ] Security Controller
- [ ] Docker Registry
- [ ] Workflow Manager
- [ ] Load Balancer
