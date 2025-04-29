#  Blue-Green Deployment on AWS EKS using ArgoCD and Argo Rollouts

---

## 🌍 Real-World Scenario

> **You're a DevOps engineer at a fast-growing SaaS company. To minimize downtime during deployments and make rollbacks seamless, you implement a **Blue-Green deployment mechanism** using **Argo Rollouts** on an EKS cluster. The app can be tested in the "Green" environment before switching live traffic from "Blue" to "Green".

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

```markdown


## 🧠 Technologies Used

- Kubernetes (Amazon EKS)
- ArgoCD
- Argo Rollouts
- Docker
- Jenkins (optional for CI)
- Bash scripting

---

## 🛠️ Step-by-Step Setup

---

### 1️⃣ Provision EKS Cluster

Use **eksctl** to create an EKS cluster:

```bash
eksctl create cluster \
  --name bluegreen-cluster \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2
```

---

### 2️⃣ Install ArgoCD and Argo Rollouts

Run the automated setup script:

```bash
bash setup.sh
```

The script does:

- Install **ArgoCD**
- Install **Argo Rollouts Controller**
- Install **Argo Rollouts CLI plugin**
- Forward ArgoCD UI on port 8080
- Forward Argo Rollouts dashboard on port 3100

---

### 3️⃣ Build and Push App Version 1

```bash
docker build -t your-repo/sample-app:1 .
docker push your-repo/sample-app:1
```

✅ `app.js` responds with `"Hello from Version 1"`

---

### 4️⃣ Deploy Blue Version

Apply rollout and service:

```bash
kubectl apply -f rollout.yaml
kubectl apply -f service.yaml
```

✅ `sample-app-active` serves Version 1.

---

### 5️⃣ Upgrade to Version 2

Switch to v2 code:

```bash
cp app-v2.js app.js
docker build -t your-repo/sample-app:2 .
docker push your-repo/sample-app:2
```

Update `rollout.yaml` to use `sample-app:2`, commit and push.  
ArgoCD detects the change ➔ Deploys Green quietly under `sample-app-preview`.

---

### 6️⃣ Test the Green Version

Port-forward preview service:

```bash
kubectl port-forward svc/sample-app-preview 8081:80
```

Visit: [http://localhost:8081](http://localhost:8081)

✅ You should see `"Hello from Version 2"`

---

### 7️⃣ Promote Green to Live

When tests pass:

```bash
kubectl argo rollouts promote sample-app
```

✅ Live traffic now switches to Version 2 (Green).

---

### 8️⃣ Rollback if Necessary

If anything goes wrong:

```bash
kubectl argo rollouts undo sample-app
```

✅ Instantly rollback to Version 1 (Blue).

---

## 📈 Argo Rollouts Dashboard Access

Run:

```bash
kubectl argo rollouts dashboard
```

Visit:

```bash
http://localhost:3100/rollouts
```

✅ Visual live status of deployments!

---

## 📋 Rollout Management CLI Commands

| Command | Purpose |
|:--------|:--------|
| `kubectl argo rollouts get rollout sample-app` | View rollout details |
| `kubectl argo rollouts promote sample-app` | Promote Green version to live |
| `kubectl argo rollouts undo sample-app` | Rollback to previous Blue version |
| `kubectl argo rollouts dashboard` | Open Rollouts visual dashboard |

---

## 🛡️ How to Check Which Version is Live

Run the script:

```bash
bash check-version.sh
```

Or manually:

```bash
kubectl get pods -l app=sample-app -o wide
kubectl port-forward svc/sample-app-active 8080:80
```

Access [http://localhost:8080](http://localhost:8080) and check which version message you see.

---

## 📈 Deployment Flow Diagram

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

## ✅ Benefits

- 🚀 Zero Downtime
- 🔥 Safe Version Testing
- 🔄 Instant Rollback
- 📈 Visual Observability
- 🔧 Full Automation Setup


---

# ✅ Final Result

This project now includes:

- EKS cluster setup
- ArgoCD install + usage
- Argo Rollouts install + usage
- Jenkinsfile for optional CI
- Rollout management
- Visual dashboard for rollout progress
- Complete upgrade simulation (v1 ➔ v2 ➔ promote ➔ undo)

---

