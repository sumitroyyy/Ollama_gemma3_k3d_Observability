This project demonstrates how to:

- Deploy **Ollama** in a Kubernetes cluster
- Expose **metrics** for Prometheus scraping
- Visualize them in **Grafana**
- Solve common **real-world issues** like exporter connectivity, scraping errors, and memory limits

Everything here is tested on **k3d (k3s)** but works on any Kubernetes cluster.

---

## ✨ Features

- Ollama model deployment in Kubernetes
- Sidecar-based Prometheus Exporter for metrics
- Annotation-based scraping for Prometheus
- Grafana dashboards for real-time observability
- Documented **problems + solutions** for production readiness

---

## 📂 Project Structure

```
ollama-monitoring/
├── ollama-deployment.yaml         # Ollama deployment with exporter sidecar
├── ollama1-pvc.yaml               # Persistent Volume Claim for model storage
├── prometheus-config-patch.yaml   # Patch for Prometheus scrape configs


```

---

## 🛠 Prerequisites

- Kubernetes cluster (k3d, Minikube, or any K8s)
- `kubectl` and `helm` CLI tools installed
- 4GB+ memory available (for the Gemma 2B model)

---

## 🚀 Step 1: Install Prometheus & Grafana via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/prometheus -n monitoring --create-namespace

# Install Grafana
helm install grafana grafana/grafana -n monitoring --create-namespace

```

Check Pods:

```bash
kubectl get pods -n monitoring

```

Port-forward UIs:

```bash
kubectl port-forward svc/prometheus-server 9090:80 -n monitoring
kubectl port-forward svc/grafana 3000:80 -n monitoring

```

Grafana → [**http://localhost:3000**](http://localhost:3000/)

- Default user/pass: `admin / admin`

---

## 📦 Step 2: Deploy Ollama with PVC

### 2.1 Persistent Storage for Models

```bash
kubectl apply -f ollama1-pvc.yaml

```

This ensures models are stored persistently and don’t need to be re-downloaded on pod restarts.

---

### 2.2 Ollama Deployment with Exporter Sidecar

```bash
kubectl apply -f ollama-deployment.yaml

```

- The main container: Runs **Ollama server**
- Sidecar container: Runs **Prometheus Exporter** that converts Ollama JSON metrics → Prometheus format

---

## 📊 Step 3: Patch Prometheus for Scraping

By default, Prometheus won’t scrape Ollama’s metrics.

```bash
kubectl patch configmap prometheus-server -n monitoring --patch-file prometheus-config-patch.yaml
kubectl rollout restart deployment prometheus-server -n monitoring

```

Check Targets → **Prometheus → Status → Targets** → should show Ollama as "UP".

---

## 🐞 Problems Faced & Fixes

Here’s a **complete list of real-world issues** and how we fixed them:

---

### 1️⃣ Container Crash Loop: Invalid `-host` Flag

**Problem:**

Ollama crashed with:

```
Error: unknown flag: --host

```

**Cause:**

The `--host` flag was deprecated.

**Fix:**

Use environment variable instead:

```yaml
env:
  - name: OLLAMA_HOST
    value: "0.0.0.0:11434"

```

---

### 2️⃣ Memory Limit Errors (OOMKilled, Exit Code 137)

**Problem:**

Model pod restarted with OOMKilled errors.

**Cause:**

Initial memory limits (1GB) too low for Gemma 2B model (1.7GB).

**Fix:**

Updated resource limits in `ollama-deployment.yaml`:

```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "4000m"

```

---

### 3️⃣ Prometheus Scrape Error: `unsupported Content-Type "application/json"`

**Problem:**

Prometheus expects text-based metrics, Ollama gives JSON.

**Solution:**

- Added **Ollama Exporter** sidecar to convert JSON → Prometheus format.
- Used correct exporter image:
    
    ```yaml
    image: lucabecker42/ollama-exporter
    args:
      - "--ollama.url=http://localhost:11434"
      - "--web.listen-address=:8000"
    
    ```
    
- Scraped `:8000/metrics` instead of Ollama API.

---

### 4️⃣ Port Mismatch: Prometheus Target Down

**Problem:**

Exporter exposed port 8000, but Prometheus annotations scraped 9090.

**Fix:**

Match both:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/path: /metrics
  prometheus.io/port: "8000"

```

---

## 📈 Step 4: Grafana Dashboards

1. Add Prometheus as a data source → `http://prometheus-server.monitoring.svc.cluster.local`
2. Import Kubernetes or Node Exporter Dashboards for instant metrics
3. Add custom panels for:
    - Memory vs Limits
    - CPU usage
    - Pod restarts
    - API latency

---

## 🧪 Useful Commands

Check pod logs:

```bash
kubectl logs -n models -l app=ollama -f

```

Check events sorted by time:

```bash
kubectl get events -n models --sort-by='.lastTimestamp'

```

Debug inside container:

```bash
kubectl exec -it -n models <pod-name> -- /bin/sh

```

---

## 📜 Final Working Setup

- **Model**: Gemma 2B
- **API**: `http://ollama-service.models:11434`
- **Prometheus Scraping**: Working via sidecar exporter
- **Grafana Dashboards**: Real-time metrics

---

## 🗝 Key Learnings

- **Exporter sidecar** pattern is best for metrics format conversion
- **Memory planning** is critical for AI models
- **Prometheus annotations** must match the actual container ports
- **Environment variables** > command-line flags for container config
