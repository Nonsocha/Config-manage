# Gitops Configuration Management


### Part 1 — Integrating Helm with ArgoCD
#### Step 1 — Create a Project Folder

On your local machine or Git repo, create a repository.

Example:

```
mkdir config-mange
cd config-manage
git init
```

Step 2 — Create a Helm Chart

Install Helm if not installed:
```
helm version
```

### Step 3  Modify values.yaml

```
nano myapp/values.yaml
```

```
replicaCount: 2

image:
  repository: nginx
  tag: latest

service:
  type: NodePort
  port: 80
```

### Step Push to GitHub
```
git add .
git commit -m "helm chart added"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/argocd-config-demo.git
git push -u origin main
```

### Step 5 — Create ArgoCD Application for Helm

Login to your ArgoCD UI.

Then create application.
Or using CLI

```
argocd app create helm-app \
--repo https://github.com/YOUR_USERNAME/argocd-config-demo.git \
--path myapp \
--dest-server https://kubernetes.default.svc \
--dest-namespace default
```

### Step 6  Sync Application
```
argocd app sync helm-app
````
```
kubectl get pods
```

### Part 2 — Using Kustomize with ArgoCD

Now we create environment configurations.

 ### STEP 2  Create Base Deployment

base/deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-kustomize
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

### Step 3  Base Kustomization File

base/kustomization.yaml

```
resources:
- deployment.yaml
- service.yaml
```

### Step 4  Dev Overlay

overlays/dev/kustomization.yaml

```
resources:
- ../../base

replicas:
- name: nginx-kustomize
  count: 2
```

### Part 3  Secrets Management in ArgoCD

#### Step 1 Install Argocd   

```
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
verify installation:

```
kubectl get pods -n argocd
```



#### Step 2 Install ArgoCD Vault Plugin (AVP)

Modify the argocd-repo-server deployment:

**Key Components Added**

- initContainer

  - Downloads AVP binary

- Volume

    - Shared between initContainer and main container

- Volume Mount

     - Makes plugin accessible to repo-server

```
volumes:
  - name: custom-tools
    emptyDir: {}

initContainers:
  - name: download-avp
    image: alpine:3.18
    command:
      - sh
      - -c
    args:
      - |
        apk add --no-cache curl
        curl -L https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v1.16.1/argocd-vault-plugin_1.16.1_linux_amd64 -o /custom-tools/argocd-vault-plugin
        chmod +x /custom-tools/argocd-vault-plugin
    volumeMounts:
      - name: custom-tools
        mountPath: /custom-tools

containers:
  - name: argocd-repo-server
    image: quay.io/argoproj/argocd:v2.11.0
    command:
      - /usr/local/bin/argocd-repo-server
    volumeMounts:
      - name: custom-tools
        mountPath: /usr/local/bin/argocd-vault-plugin
        subPath: argocd-vault-plugin
```

### Step 3: Verify Plugin Installation

```
kubectl exec -it -n argocd deploy/argocd-repo-server -- argocd-vault-plugin version
```

### Step 4: Create Kubernetes Secret
```
kubectl create secret generic my-secret \
  --from-literal=password=supersecret \
  -n argocd
```

### Step 5: Create Manifest with Secret Reference

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: argocd
data:
  password: <path:kubernetes://argocd/my-secret#password>
```
### Step 6: Push Manifest to GitHub

```
git add
git commit -m 
git push
```

### Step 6: Push Manifest to GitHub

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: avp-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/Config-manage.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```
kubectl apply -f application.yaml
```

#### Step 8 Verify secret injection:
```
kubectl get configmap my-config -n argocd -o yaml
```
