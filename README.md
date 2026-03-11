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
