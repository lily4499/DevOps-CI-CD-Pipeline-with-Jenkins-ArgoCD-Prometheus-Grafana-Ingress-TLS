
# 🚀 DevOps CI/CD Pipeline with Jenkins, ArgoCD, Prometheus, Grafana, Ingress & TLS

## 📌 Objective

Automate the end-to-end deployment of a containerized application from GitHub to AWS EKS using:

- ✅ Jenkins CI/CD
- ✅ ArgoCD GitOps
- ✅ Prometheus/Grafana for Monitoring
- ✅ Ingress with TLS via Cert-Manager
- ✅ DNS via Namecheap + DigitalOcean

---

## 🗂️ Project Structure

```
/home/lilia/VIDEOS/devops-argocd-pipeline/
├── app/                             # App source repo
│   ├── app.js
│   ├── package.json
│   ├── Dockerfile
│   └── Jenkinsfile                  # Jenkinsfile for CI: build & push image
│
├── manifests/                       # Kubernetes manifest repo
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── Jenkinsfile                 # Jenkinsfile for CD: update image tag
│
├── ingress-configs/                # Optional shared or backup ingress configs
│   ├── issuer.yaml
│   └── ingress.yaml
│
├── monitoring-configs/             # Optional helm values for monitoring
│   ├── prometheus-values.yaml
│   └── grafana-values.yaml

```

---

## 🏁 Phase 1: Build and test the Docker Image Locally

```bash
docker build -t devops-app:local .
docker run -d -p 3000:3000 --name devops-app-test devops-app:local
curl http://localhost:3000
docker stop devops-app-test
docker rm devops-app-test

```
> Visit http://localhost:3000
> Now your code is tested locally, and ready to be integrated with Jenkins for CI/CD.

---

## 🏗️ Phase 2: Jenkins CI/CD Pipeline

### ✅ Job 1 - Jenkinsfile-ci

```groovy
pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        sh 'docker build -t laly9999/app:${BUILD_NUMBER} .'
      }
    }
    stage('Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
          sh 'echo $PASS | docker login -u $USER --password-stdin'
          sh 'docker push laly9999/app:${BUILD_NUMBER}'
        }
      }
    }
    stage('Trigger CD') {
      steps {
        build job: 'update-k8s-manifests', parameters: [
          string(name: 'IMAGE_TAG', value: "${BUILD_NUMBER}")
        ]
      }
    }
  }
}
```

---

### ✅ Job 2 - Jenkinsfile-cd

```groovy
pipeline {
  agent any
  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'latest')
  }
  stages {
    stage('Checkout') {
      steps {
        git url: 'https://github.com/your-user/k8s-manifests.git', branch: 'main', credentialsId: 'github-creds'
      }
    }
    stage('Update Image Tag') {
      steps {
        sh '''
        sed -i 's|image: laly9999/app:.*|image: laly9999/app:${IMAGE_TAG}|' deployment.yaml
        git config user.email "ci@jenkins.com"
        git config user.name "Jenkins"
        git commit -am "Update image tag to ${IMAGE_TAG}"
        git push
        '''
      }
    }
  }
}

```
### Push Local Code to GitHub
📦 Push the App Repo:
```bash
cd devops-argocd-pipeline/app
git init
git remote add origin https://github.com/your-username/devops-node-app.git
git add .
git commit -m "Initial commit - Node.js app with Dockerfile"
git branch -M main
git push -u origin main
```
📦 Push the Manifest Repo:
```bash
cd ../manifests
git init
git remote add origin https://github.com/your-username/devops-k8s-manifests.git
git add .
git commit -m "Initial commit - Kubernetes manifests"
git branch -M main
git push -u origin main
```

---

## ☁️ Phase 3: EKS Setup

```bash
aws eks --region us-east-1 update-kubeconfig --name <EKS_CLUSTER_NAME>
kubectl get nodes
```

---

## 🚀 Phase 4: Install ArgoCD

```bash
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 🔐 Access ArgoCD UI

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Go to: [http://localhost:8080](http://localhost:8080)  
Login with:
- Username: `admin`
- Password: from the command above

### 🔁 CLI Login & Sync

```bash
argocd login localhost:8080
argocd account update-password
```

### 🔧 Create ArgoCD App

```bash
argocd app create myapp \
  --repo https://github.com/your-user/k8s-manifests.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev
```

---

## 📈 Phase 5: Prometheus & Grafana

### 🟡 Prometheus

```bash
kubectl create ns prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install my-prometheus prometheus-community/prometheus -n prometheus
kubectl port-forward -n prometheus deploy/my-prometheus-server 9090
```
> This makes Prometheus UI accessible at:📍 http://localhost:9090

### 🔵 Grafana

```bash
kubectl create ns grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm install my-grafana grafana/grafana -n grafana
kubectl port-forward svc/my-grafana 3000 -n grafana
kubectl get secret --namespace grafana my-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
>Login at: http://localhost:3000
> Username: admin
> Password: (from command above)

### 📦  Enable Node-Level Metrics    
Prometheus Helm chart installs Node Exporter as a DaemonSet.  
No extra steps are needed, but confirm:  
```bash
kubectl get daemonset -n prometheus | grep node-exporter
```
### 🔁 Enable Kubernetes Metrics
Ensure kube-state-metrics is installed (comes with Prometheus chart):  
```bash
kubectl get deploy -n prometheus | grep kube-state-metrics
#If not present, install separately:
helm install kube-state-metrics prometheus-community/kube-state-metrics -n prometheus
```
### How to Verify Prometheus Targets Are Up
> Go to: http://localhost:9090,
> Click on:➡️ “Status” → “Targets”
## ✅ What You Should See in Prometheus Targets

After visiting [http://localhost:9090/targets](http://localhost:9090/targets), you should see the following targets listed as **UP**:

| Target Name         | Expected Role                          | Status |
|---------------------|----------------------------------------|--------|
| `node-exporter`     | Collects CPU, memory, disk, load       | ✅ UP   |
| `kube-state-metrics`| Collects Kubernetes object metrics     | ✅ UP   |
| `prometheus-server` | Self-monitoring of Prometheus itself   | ✅ UP   |

###  Import Grafana Dashboards
In Grafana UI:    
 - Go to “Dashboards” → “Import”    
 - Use the following dashboard IDs:  
   -  Node Exporter Full: 1860  
   - Kubernetes Cluster Monitoring: 315  
Select Prometheus as the data source.  

---

## 🌐 Phase 6: Ingress & TLS with Cert-Manager

### 📌 Ingress-NGINX

```bash
kubectl create ns ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install my-nginx ingress-nginx/ingress-nginx -n ingress-nginx
```

### 🌍 Point Domain to Load Balancer

```bash
kubectl get svc -n ingress-nginx
# Copy EXTERNAL-IP and map to Namecheap/DigitalOcean DNS
```

---

### 🔒 Cert-Manager Installation

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.1/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm install my-cert jetstack/cert-manager --namespace cert-manager
```

### 📄 issuer.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-nginx
spec:
  acme:
    email: konissil@yahoo.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-nginx-private-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

### 📄 ingress.yaml (TLS Enabled)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-prod
  annotations:
    cert-manager.io/issuer: letsencrypt-nginx
spec:
  tls:
    - hosts:
        - app.lilianedevops.online
      secretName: letsencrypt-nginx
  rules:
    - host: app.lilianedevops.online
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: amazon-service
                port:
                  number: 3000
  ingressClassName: nginx
```

---

## 🔁 Phase 7: GitHub Webhook Integration

1. Go to your GitHub app repo → **Settings > Webhooks**
2. Add Webhook:
   - **Payload URL:** `http://<jenkins-server>/github-webhook/`
   - **Content Type:** `application/json`
   - **Event:** `Just the push event`
3. Push code → Jenkins CI → Jenkins CD → ArgoCD → EKS

---

## ✅ Final Test: Full Automation Flow

- 🔄 Push new commit to GitHub
- ⚙️ Jenkins builds & pushes image
- 📦 Jenkins updates deployment manifest
- 🚀 ArgoCD syncs new image to EKS
- 🌍 App is live at: `https://app.lilianedevops.online`
- 📊 Monitor using Prometheus & Grafana

---

## 🧠 Credits

Built with ❤️ by [Liliane DevOps](https://github.com/lily4499)


---

---

## 📊 Monitor These Metrics with Prometheus & Grafana

This section outlines the key metrics you should monitor in your Kubernetes environment, split into two main categories:

- 🖥️ Node-Level (Infrastructure) Metrics
- ☸️ Kubernetes Cluster Metrics

These are visualized in Grafana using data scraped by Prometheus.

---

### 🖥️ Node Metrics (Infrastructure)

These metrics help you monitor the physical or virtual machines (nodes) running your Kubernetes cluster. They are collected by **Node Exporter**.

| Panel        | What It Shows                   | Why It Matters                              |
|--------------|----------------------------------|---------------------------------------------|
| CPU usage    | Usage per node (real-time CPU %) | Detects CPU saturation and uneven workloads |
| Memory usage | RAM utilization per node         | Identifies memory leaks or OOM issues       |
| Disk I/O     | Disk read/write activity         | Tracks disk bottlenecks                     |
| Load avg     | System load over time            | Checks overall node load & stress           |

🛠 **Grafana Setup:**
- Create a dashboard and use Prometheus queries like:
  - `rate(node_cpu_seconds_total{mode!="idle"}[5m])`
  - `node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes`
  - `rate(node_disk_read_bytes_total[5m])`
  - `node_load1`

📥 **Optional Import:**  
Use Grafana Dashboard ID `1860` (Node Exporter Full)

---

### ☸️ Kubernetes Cluster Metrics

These metrics monitor the health and performance of Kubernetes workloads. They are collected by **kube-state-metrics**.

| Panel              | What It Shows                         | Why It Matters                                      |
|--------------------|----------------------------------------|-----------------------------------------------------|
| Pod restarts       | Number of restarts per pod             | Identifies crashing containers or instability       |
| Deployment health  | Available vs desired replicas          | Ensures all app instances are running as expected   |
| Container resources| CPU/Memory usage per container         | Optimizes resource allocation and limits            |
| Pod status         | Running, Pending, Failed, etc.         | Gives real-time view into pod lifecycle states      |

🛠 **Grafana Setup:**
- Add panels using Prometheus queries like:
  - `kube_pod_container_status_restarts_total`
  - `kube_deployment_status_replicas_unavailable`
  - `container_memory_usage_bytes`
  - `kube_pod_status_phase`

📥 **Optional Import:**  
Use Grafana Dashboard ID `315` (Kubernetes Cluster Monitoring)

---

### 📌 Summary

By combining these dashboards, you can:

- Detect performance issues in infrastructure or apps
- Set alerts based on thresholds (e.g., too many restarts)
- Gain full observability into your Kubernetes environment
```

---


