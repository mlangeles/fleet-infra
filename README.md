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
kubectl -n $NAMESPACE port-forward --address 0.0.0.0 svc/argocd-server 8080:443

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

### Harbor 

Harbor is an open source registry that secures artifacts with policies and role-based access control, ensures images are scanned and free from vulnerabilities, and signs images as trusted. 

Assumptions:
- Running Kubernetes cluster

```bash
# Source
# https://vmwire.com/2022/03/04/deploy-harbor-registry-with-helm-and-expose-with-ingress/
# https://github.com/goharbor/harbor-helm

# Add Harbor Repo
helm repo add harbor https://helm.goharbor.io

# Harbor Repo Validation
helm search repo harbor

# Harbor Helm Installation
helm install \
  --namespace harbor \
  --create-namespace \
  operator harbor/harbor

# Harbor Installtion Verification
kubectl get all -n harbor

# Harbor Credentials
# User: admin
# Pass: Harbor12345

https://core.harbor.domain/

# Harbor Proxy Dashboard
kubectl -n harbor port-forward --address 0.0.0.0 svc/operator-harbor-portal 31080:80

kubectl exec -ti buildah-pod -- bash

buildah login core.harbor.domain --tls-verify=false -u admin -p Harbor12345

buildah pull --tls-verify=false core.harbor.domain/dockerhub/alpine:latest
```

### Argo Workflow

Argo Workflows is an open source container-native workflow engine for orchestrating parallel jobs on Kubernetes.

Assumptions:
- Running Kubernetes cluster

```bash
export ARGOWORKFLOW=argoworkflow
# Add Argo Workflow Helm Repository
helm repo add argo https://argoproj.github.io/argo-helm

# Argo Workflow Helm Chart Installation
helm -n argocd install $ARGOWORKFLOW argo/argo-workflows --version 0.40.10

# Argo Workflow Store Installation Template to File 
helm template $ARGOWORKFLOW argo/argo-workflows --version 0.40.10 > test.yaml

# Argo Workflow Helm Chart Uninstallion from Namespace $ARGOWORKFLOW
helm -n argocd uninstall $ARGOWORKFLOW

#Argo Workflow Port-forward
kubectl -n argocd port-forward --address 0.0.0.0 svc/helm-argoworkflow-argo-workflows-server 2746:2746

#Generate Argo Workflow API Token
kubectl -n argocd create token helm-argoworkflow-argo-workflows-server 

#Source: https://github.com/argoproj/argo-workflows/blob/main/examples/buildkit-template.yaml
```



# TODO
- [x] Network Controller 
- [ ] Storage Controller
- [ ] Security Controller
- [x] Docker Registry
- [ ] Workflow Manager
```bash
# Add the following into an argo workflow template
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Pod
metadata:
  name: buildah-pod
spec:
  hostAliases:
  - ip: "192.168.159.129"
    hostnames:
    - "core.harbor.domain"
  containers:
  - name: buildah-container
    image: quay.io/buildah/stable
    command: ["bash"]
    tty: true
    volumeMounts:
    - name: buildah-config
      mountPath: /mnt/containers/
    securityContext:
      privileged: true
  volumes:
  - name: buildah-config
    configMap:
      name: buildah-config
EOF

```
- [ ] Load Balancer
