
         

# 🚀 GitOps with ArgoCD and Argo Image Updater on kind Kubernetes Cluster:
---------------------------------------------------------------------------

## 🎯 Objective:

Implement a fully working GitOps pipeline using ArgoCD and Argo Image Updater on a local Kubernetes cluster (created with `kind`) to automatically deploy and update an NGINX application via GitHub.

---

## 🧰 Tools Used:

* kind (Kubernetes in Docker)
* kubectl
* ArgoCD
* Argo Image Updater
* GitHub
* Docker

---


## ✅ Prerequisites

Make sure these tools are installed:

```bash
kind version
kubectl version --client
argocd version
docker --version
git --version
```

![Image](https://github.com/user-attachments/assets/6177f917-070b-4325-bcd3-e4dd73e43fa1)

Create a GitHub repository, e.g., `https://github.com/<your-username>/gitops-demo`

---

## 🪜 Step-by-Step Guide

### 🔹 Step 1: Create a Kind Cluster

```bash
kind create cluster --name gitops-demo
```

** Output:**

```
Creating cluster "gitops-demo" ...
Cluster creation complete.
```

Check the nodes:

```bash
kubectl get nodes
```
![Image](https://github.com/user-attachments/assets/68a68e05-45b8-4b51-83c3-1501aac9228b)

**Expected Output:**

```
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   1m      v1.27.0
```

---

### 🔹 Step 2: Install ArgoCD in the Cluster

```bash
kubectl create namespace argocd
```

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
![Image](https://github.com/user-attachments/assets/587a962f-f586-4be3-bb8a-3c18d7c5119e)

Wait for all pods to be ready:

```bash
kubectl get pods -n argocd -w
```

![Image](https://github.com/user-attachments/assets/8c2c1b68-817f-4528-b38c-c0fa18c47e3e)

---

### 🔹 Step 3: Expose ArgoCD UI (Port Forward)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
![Image](https://github.com/user-attachments/assets/b77ed496-a778-4f44-8b14-26f48a2ff020)


Access ArgoCD UI at: [https://localhost:8080](https://localhost:8080)

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

![Image](https://github.com/user-attachments/assets/530eb621-ea40-494c-869a-b5e7029fcce8)


Login using CLI:

```bash
argocd login localhost:8080 --username admin --password <your-password> --insecure
```

**Sample Output:**

```
'admin' logged in successfully
```

---

### 🔹 Step 4: Create Your GitHub Repo & Push App Manifests

#### 📁 Folder Structure:

```
gitops-demo/
├── nginx/
│   ├── deployment.yaml
│   └── service.yaml
├── README.md
```

#### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
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
          image: nginx:1.25
          ports:
            - containerPort: 80
```

#### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

Push the code to GitHub:

```bash
git init
git remote add origin https://github.com/<your-username>/gitops-demo.git
git add .
git commit -m "Initial commit"
git push -u origin main
```

---

### 🔹 Step 5: Create ArgoCD Application

```bash
argocd app create nginx-app \
  --repo https://github.com/<your-username>/gitops-demo.git \
  --path nginx \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

Enable auto-sync:

```bash
argocd app set nginx-app --sync-policy automated
```

Sync manually if needed:

```bash
argocd app sync nginx-app
```

Otherway to create Argocd Application:
---------------------------------------

✅ Step-by-Step: Create an Argo CD App from the UI
---------------------------------------------------
Access the Argo CD UI:
-----------------------
Open your browser and go to:
https://localhost:8080
(since you're doing port-forwarding with kubectl port-forward svc/argocd-server -n argocd 8080:443)

Login

Username: admin

Password: (from the decoded argocd-initial-admin-secret)

Click “+ NEW APP” (top-left corner)

Fill in the Application Form:
------------------------------
Field-  Value:
--------------
Application Name-nginx-app

Project-default

Sync Policy	-Manual (or Automatic, your choice)

Repository URL- https://github.com/<your-username>/gitops-demo.git

Revision-HEAD (or a specific branch/tag)

Path-gitops-demo/nginx (this is the folder with Kubernetes manifests inside the repo)

Cluster URL-https://kubernetes.default.svc (default in-cluster config)

Namespace-  default
---
Click “Create”

🔄 (Optional) Sync the App:
----------------------------
After creation, you’ll see the app listed in the UI.

Click on the nginx-app

Click SYNC → Synchronize to deploy it to your cluster

**Sample Output:**

```
application 'nginx-app' created
```
![Image](https://github.com/user-attachments/assets/682c1720-8c54-45af-beac-50b4b3746a11)


Check status:

```bash
argocd app get nginx-app
```

![Image](https://github.com/user-attachments/assets/34ef7168-4667-4d2b-a6d7-95841395e8fe)

---

### 🔹 Step 6: Verify Deployment

```bash
kubectl get all
```

**Sample Output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
deployment.apps/nginx              1/1     Running   0          1m
service/nginx-service              NodePort 80:3xxxx/TCP        1m
```

![Image](https://github.com/user-attachments/assets/084f6fd4-7bba-468d-818f-cc5473fc25b4)

---

### 🔹 Step 7: Install Argo Image Updater

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/master/manifests/install.yaml

```

Wait for deployment:

```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-image-updater
```

![Image](https://github.com/user-attachments/assets/dd50782e-b488-4592-a72d-68ecf7e3cc94)


---

### 🔹 Step 8: Enable Auto Image Update via Annotations

```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: nginx=nginx:latest  # Add tag
    argocd-image-updater.argoproj.io/nginx.update-strategy: latest
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
    argocd-image-updater.argoproj.io/git-commit-user-name: Auto Updater
    argocd-image-updater.argoproj.io/git-commit-user-email: auto@updater.com
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://github.com/SuryaSJV/GitOps-project-10-.git
    targetRevision: main
    path: gitops-demo/nginx/
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
```

---
## note : IF YOU USE GITHUB PUBLIC REPO YOU DONT NEED TO DO "Step 9" 
### 🔹 Step 9: Configure GitHub Credentials Secret

Generate a GitHub PAT (Personal Access Token) with `repo` scope.

Create secret:

```bash
kubectl -n argocd create secret generic git-creds \
  --from-literal=username='<your-github-username>' \
  --from-literal=password='<your-pat>' \
  --type=kubernetes.io/basic-auth
```

Label the secret:

```bash
kubectl -n argocd label secret git-creds "argocd.argoproj.io/secret-type=repo-creds"
```

---

### 🔹 Step 10: Validate Image Auto-Updater

Check logs:

```bash
kubectl logs -n argocd deployment/argocd-image-updater
```

**Sample Output:**

```
INFO[0000] Processing application nginx-app
INFO[0000] Latest tag for image nginx is 1.25.3
INFO[0000] Updated image tag to nginx:1.25.3 in deployment.yaml
```

Push updated image to GitHub → ArgoCD auto-syncs → New version deployed.

---

## 🎉 SUCCESS! Project Completed

![Image](https://github.com/user-attachments/assets/6781f145-0586-447c-a485-2a60f710c088)

![Image](https://github.com/user-attachments/assets/9653705e-e255-4b0e-80cc-f37d98810c6b)



You now have a:

* GitOps-enabled Kubernetes cluster using ArgoCD
* Auto-sync from GitHub repository
* Auto-image updates via Argo Image Updater

---

![Image](https://github.com/user-attachments/assets/ad9e611e-582c-4ec3-a119-2f4906b96b40)


![Image](https://github.com/user-attachments/assets/516a58cc-0e6b-4691-9b63-fdad1de9551f)





⚠️ Key Point: Kind runs inside a Docker container
When you expose a service using NodePort, that port is exposed inside the Docker container, not directly to your host machine unless you explicitly publish the port when creating the kind cluster.




✅ Option 1: Best Practice — Use kubectl port-forward
Quick and effective for local dev/testing:

```
kubectl port-forward svc/nginx-service 8888:80
```
Then access it in your browser at:
```
http://localhost:8888
```
![Image](https://github.com/user-attachments/assets/db4b5b4c-f0e8-4174-bcea-6e9eae0b41e9)


As we knew The Deployed Application is sample NGINX application, it would look like 

![Image](https://github.com/user-attachments/assets/0795c03c-70c2-47c5-bb1d-e2eceaec5b22)





## Happy GitOpssify to reduce the manual effort 

   since instead of using shell script the image tag in deployment.yaml file we just had used "Argo Image Updater" this way we can lowerdown the manual intervenstion to upgrade the newer version our application.

### using Argo Image Updater makes a big differnece as below

![Image](https://github.com/user-attachments/assets/6fd1b094-39c5-418c-a174-4bde1bcbf14a)