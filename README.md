# Self Managed Argo CD - App of Everything

**Table of Contents**

- [Introduction](#introduction)
- [Clone Repository](#clone-repository)
- [Create Local Kubernetes Cluster](#create-local-kubernetes-cluster)
- [Git Repository Hierarchy](#git-repository-hierarchy)
- [Create App Of Everything Pattern](#create-app-of-everything-pattern)
- [Intall Argo CD Using Helm](#intall-argo-cd-using-helm)
- [Demo With Sample Application](#demo-with-sample-application)
- [Cleanup](#cleanup)

# Introduction
This project aims to install a self-managed Argo CD using the App of App pattern. 

# Clone Repository
Clone gokul0815/argocd repository to your local device.
```
git clone https://github.com/William-eng/argocd.git
```
# Create Rancher Kubernetes Cluster
For Ec2-instance:
First install rancher on the master mode, use this [documentation](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade)

## Install Master
```
curl -sfL https://get.k3s.io | sh -s - --disable traefik --write-kubeconfig-mode 644 --node-name k3s-master-01
```

## Install Worker
Grab token from the master node to be able to add worked nodes to it:

```
cat /var/lib/rancher/k3s/server/node-token
```
Install k3s on the worker node and add it to our cluster:


```
curl -sfL https://get.k3s.io | K3S_NODE_NAME=k3s-worker-01 K3S_URL=https://<IP>:6443 K3S_TOKEN=<TOKEN> sh - 

```

- ![Image1](https://github.com/user-attachments/assets/7a004073-e977-4011-9746-b67742379e12)


# OPTIONAL - Install nginx ingress controller
Website: https://kubernetes.github.io/ingress-nginx/

```

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.1/deploy/static/provider/cloud/deploy.yaml

```

# Git Repository Hierarchy
Folder structure below is used in this project. You are free to change it.
```
argocd/
├── argocd-appprojects      # stores ArgoCD App Project's yaml files
├── argocd-apps             # stores ArgoCD Application's yaml files
├── argocd-install          # stores Argo CD installation files
│ ├── argo-cd               # argo/argo-cd helm chart
│ └── argo.tf               # terraform file for argo-cd chart
│ └── install.sh            # custom values.yaml for argo-cd chart
```

# Create App Of Everything Pattern

Open *argocd-install/install.sh* with your favorite editor and modify related values.
```
vi argocd-install/install.sh
```
Or update it with your values.
```
cat << EOF > argocd-install/argo.tf
---
global:
  image:
    tag: "v2.6.6"

dex:
  enabled: false

server:
  extraArgs:
    - --insecure
EOF
```

# Intall Argo CD Using Helm
Go to argocd directory.
```
cd argocd/argocd-install/
```

Intall Argo CD with the commands

```
terraform init
terraform plan
terraform apply --auto-approve
```
- ![Image2](https://github.com/user-attachments/assets/dbf8837d-63e4-4c69-8a7b-969697270fdc)
- ![Image3](https://github.com/user-attachments/assets/b2a26bb0-2dc8-48ad-a091-6b4a2c8035a5)


Wait until all pods are running.
```
kubectl -n argocd get pods

NAME                                            READY   STATUS    RESTARTS
argocd-application-controller-bcc4f7584-vsbc7   1/1     Running   0       
argocd-dex-server-77f6fc6cfb-v844k              1/1     Running   0       
argocd-redis-7966999975-68hm7                   1/1     Running   0       
argocd-repo-server-6b76b7ff6b-2fgqr             1/1     Running   0       
argocd-server-848dbc6cb4-r48qp                  1/1     Running   0
```

Get initial admin password.
```
kubectl -n argocd get secrets argocd-initial-admin-secret \
    -o jsonpath='{.data.password}' | base64 -d
```

Forward argocd-server service port 80 to localhost:8080 using kubectl.
```
kubectl -n argocd port-forward service/argocd-server 8080:80
```

- ![Image6](https://github.com/user-attachments/assets/f7d0987f-06ab-4cf9-8ab9-daf3c7b08953)


Browse http://<public-ip-address>:8080 and login with initial admin password.

# Demo With Sample Application
Create an application project definition file called *sample-project*.
```
cat << EOF > argocd-appprojects/sample-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: sample-project
  namespace: argocd
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  destinations:
  - namespace: sample-app
    server: https://kubernetes.default.svc
  orphanedResources:
    warn: false
  sourceRepos:
  - '*'
EOF
```

Push changes to your repository.
```
git add argocd-appprojects/sample-project.yaml
git commit -m "Create sample-project"
git push
```

Create a saple applicaiton definition yaml file called *sample-app* under argocd-apps.
```
cat << EOF >> argocd-apps/sample-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
spec:
  destination:
    namespace: sample-app
    server: https://kubernetes.default.svc
  project: sample-project
  source:
    path: sample-app/
    repoURL: https://github.com/William-eng/argocd.git
    targetRevision: HEAD
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    automated:
      selfHeal: true
      prune: true
EOF
```

Push changes to your repository.
```
git add argocd-apps/sample-app.yaml
git commit -m "Create application"
git push

```
## Deploying Application via Helm to Argo CD
Let’s create an application on ArgoCD

```
argocd login <argo-cd-server-address> --username admin --password <your-admin-password> --insecure

argocd app create helm-guestbook --repo https://github.com/William-eng/argocd.git --path helm-argocd1 --dest-server https://kubernetes.default.svc --dest-namespace default

```
- ![Image7](https://github.com/user-attachments/assets/1b7d0c3d-7a0e-4255-9bdf-348d8147eb23)


You can now see that the health status of our Service is Healthy and Deployment is Progressing. The Deployment will take some time and becomes Healthy.

You can verify the created application on Argo CD by going to the UI.

- ![Image8](https://github.com/user-attachments/assets/10f4d584-d6e7-4ff1-9cdd-d585cc3ef079)
- ![Image9 ](https://github.com/user-attachments/assets/a9e25070-5593-483f-9071-c1076ab16848)



# Cleanup
Remove application and applicaiton project.
```
rm -f argocd-apps/sample-app.yaml
rm -f argocd-appprojects/sample-project.yaml
git rm argocd-apps/sample-app.yaml
git rm argocd-appprojects/sample-project.yaml
git commit -m "Remove app and project."
git push
```
