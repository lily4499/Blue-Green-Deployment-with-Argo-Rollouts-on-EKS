#  Blue-Green Deployment on AWS EKS using ArgoCD and Argo Rollouts

---

## 🌍 Real-World Scenario

> You're a DevOps engineer at a fast-growing SaaS company. To minimize downtime during deployments and make rollbacks seamless, you implement a **Blue-Green deployment mechanism** using **Argo Rollouts** on an EKS cluster. The app can be tested in the "Green" environment before switching live traffic from "Blue" to "Green".

---

# 📂 Directory Structure

```bash
blue-green-eks/
├── Dockerfile               # Build container image
├── app.js                   # Version 1 (Hello v1)
├── app-v2.js                # Version 2 (Hello v2)
├── Jenkinsfile              # (Optional) CI: Build & Push Docker image
├── rollout.yaml             # Argo Rollouts resource with Blue-Green strategy
├── service.yaml             # Kubernetes Services (active + preview)
├── check-version.sh         # Bash script to check live version
├── setup.sh                 # One-click installer for ArgoCD, Argo Rollouts
├── README.md                # Full updated documentation
```
---


## file-setup.py

```python
import os

# Base directory
base_dir = "/home/lilia/VIDEOS/blue-green-eks"

# Files with their content
files = {
    "Dockerfile": """
FROM node:18
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD [ "node", "app.js" ]
""",

    "app.js": """
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.send('Hello from Version 1 (Blue)!');
});

app.listen(port, () => {
  console.log(`App running on port ${port}`);
});
""",

    "app-v2.js": """
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.send('Hello from Version 2 (Green)!');
});

app.listen(port, () => {
  console.log(`App running on port ${port}`);
});
""",

    "Jenkinsfile": """
pipeline {
    agent any
    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t your-repo/sample-app:$BUILD_NUMBER .'
            }
        }
        stage('Push Docker Image') {
            steps {
                withDockerRegistry([credentialsId: 'dockerhub-credentials', url: '']) {
                    sh 'docker push your-repo/sample-app:$BUILD_NUMBER'
                }
            }
        }
    }
}
""",

    "rollout.yaml": """
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: sample-app
spec:
  replicas: 2
  strategy:
    blueGreen:
      activeService: sample-app-active
      previewService: sample-app-preview
      autoPromotionEnabled: false
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: your-repo/sample-app:1
        ports:
        - containerPort: 3000
""",

    "service.yaml": """
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app-active
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app-preview
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 3000
""",

    "check-version.sh": """
#!/bin/bash

echo "\ud83d\udd0d Checking current live app version:"
kubectl get pods -l app=sample-app -o=jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.containers[*].image}{"\\n"}{end}'
""",

    "setup.sh": """
#!/bin/bash

set -e

echo "\ud83d\ude80 Installing ArgoCD..."
kubectl create namespace argocd || true
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

echo "\u23f3 Waiting for ArgoCD pods to come up..."
sleep 60

echo "\ud83d\ude80 Installing Argo Rollouts Controller..."
kubectl create namespace argo-rollouts || true
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

echo "\ud83d\ude80 Installing Argo Rollouts CLI Plugin..."
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

echo "\u2705 Setup complete!"

echo "\ud83d\udd17 Forwarding ArgoCD server to localhost:8080..."
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

sleep 5

echo "\ud83d\udd17 Forwarding Argo Rollouts Dashboard to localhost:3100..."
kubectl argo rollouts dashboard &
"""
}

# Create base directory
os.makedirs(base_dir, exist_ok=True)

# Write each file
for filename, content in files.items():
    file_path = os.path.join(base_dir, filename)
    with open(file_path, 'w') as f:
        f.write(content.strip())

print(f"\n✅ All files created successfully under {base_dir}")


```

---


## 🧠 Technologies Used

- Kubernetes (Amazon EKS)
- ArgoCD
- Argo Rollouts
- Docker
- Jenkins (optional for CI)
- Bash scripting

---

# 🛠️ Full Step-by-Step Setup

---

## 1️⃣ Provision EKS Cluster

Use **eksctl** to create a Kubernetes cluster on AWS:

```bash
eksctl create cluster \
  --name bluegreen-cluster \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2
```

✅ A working EKS cluster will be created.

---

## 2️⃣ Install ArgoCD and Argo Rollouts

Run the automated setup script:

```bash
bash setup.sh
```

✅ This script:
- Installs **ArgoCD**
- Installs **Argo Rollouts Controller**
- Installs **Argo Rollouts CLI Plugin**
- Port-forwards ArgoCD UI on `localhost:8080`
- Opens Argo Rollouts dashboard at `localhost:3100`

---

## 3️⃣ Create Application in ArgoCD UI

1. Visit [https://localhost:8080](https://localhost:8080).

2. Login:
   - Username: `admin`
   - Password:

     ```bash
     kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
     ```

3. Click **`+ New App`**.

4. Fill out the fields:

| Field | Value |
|:------|:------|
| **Application Name** | `sample-app` |
| **Project** | `default` |
| **Sync Policy** | Manual or Automatic |
| **Repository URL** | Your GitHub Repo (example: `https://github.com/your-username/blue-green-eks.git`) |
| **Revision** | `HEAD` |
| **Path** | `/` |
| **Cluster** | `https://kubernetes.default.svc` |
| **Namespace** | `default` |

5. Click **Create**.

✅ ArgoCD will now automatically sync your rollout.yaml and service.yaml.

---

## 4️⃣ Build and Push App Version 1 (Blue)

```bash
docker build -t your-repo/sample-app:1 .
docker push your-repo/sample-app:1
```

✅ App version 1 (`app.js`) responds with `"Hello from Version 1"`.

---

## 5️⃣ Deploy Blue Version

Sync the Application inside ArgoCD (click **Sync** button).  
Or manually apply:

```bash
kubectl apply -f rollout.yaml
kubectl apply -f service.yaml
```

✅ `sample-app-active` Service is serving Version 1 (Blue).

---

## 6️⃣ Upgrade to Version 2 (Green)

Switch code to version 2:

```bash
cp app-v2.js app.js
docker build -t your-repo/sample-app:2 .
docker push your-repo/sample-app:2
```

Update `rollout.yaml`:

```yaml
containers:
- name: sample-app
  image: your-repo/sample-app:2
```

Push this update to GitHub.

✅ ArgoCD automatically detects changes and deploys Green pods behind `sample-app-preview`.

---

## 7️⃣ Test the Green Version

Port-forward preview service:

```bash
kubectl port-forward svc/sample-app-preview 8081:80
```

Visit:  
[http://localhost:8081](http://localhost:8081)

✅ You should see `"Hello from Version 2 (Green)!"`

✅ Live users are **still on Version 1 (Blue)**.

---

## 8️⃣ Promote Green to Live

When testing is successful:

```bash
kubectl argo rollouts promote sample-app
```

✅ Live production traffic now switches to Version 2 (Green) **without downtime**.

✅ Argo Rollouts updates `sample-app-active` service to point to Green pods.

---

## 9️⃣ Rollback if Necessary

If issues occur even after promotion:

```bash
kubectl argo rollouts undo sample-app
```

✅ Instantly rollback to previous Version 1 (Blue) without downtime.

---

# 📈 Argo Rollouts Dashboard Access

Launch dashboard:

```bash
kubectl argo rollouts dashboard
```

Open browser at:  
[http://localhost:3100/rollouts](http://localhost:3100/rollouts)

✅ You can visually monitor Blue-Green deployments, promotion, and rollback events.

---

# 📋 Rollout Management CLI Commands

| Command | Purpose |
|:--------|:--------|
| `kubectl argo rollouts get rollout sample-app` | View rollout status |
| `kubectl argo rollouts promote sample-app` | Promote Green version |
| `kubectl argo rollouts undo sample-app` | Rollback to previous Blue version |
| `kubectl argo rollouts dashboard` | Open visual dashboard |

---

# 🛡️ How to Check Which Version is Live

Run the helper script:

```bash
bash check-version.sh
```

Or manually:

```bash
kubectl get pods -l app=sample-app -o wide
kubectl port-forward svc/sample-app-active 8080:80
```

Visit:  
[http://localhost:8080](http://localhost:8080)

✅ You can verify if users are seeing Version 1 or Version 2.

---

# 📈 Deployment Flow Diagram

```mermaid
flowchart TD
    A[Blue (v1) Live] -->|Deploy Green (v2)| B[Green Pods Created]
    B -->|Internal Test| C{Pass?}
    C -- Yes --> D[Promote Green to Active]
    C -- No --> E[Undo Rollout]
    D --> F[Green (v2) Live]
    E --> A
```

---

# ✅ Benefits

- 🚀 Zero Downtime Deployment
- 🔥 Safe Version Testing (before switch)
- 🔄 Instant Rollbacks
- 📈 Visual Observability via Dashboard
- 🔧 GitOps-ready with ArgoCD Automation

---

# 📜 License

MIT License

---

# 🌟 Final Notes

✅ This project demonstrates real-world **Progressive Delivery (Blue-Green)** best practices used by companies like Netflix, Shopify, Amazon.  
✅ It's production-grade, safe, observable, and GitOps-driven with ArgoCD + Argo Rollouts.

---

