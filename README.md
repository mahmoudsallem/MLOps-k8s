# Kubernetes Course Tasks (MLOps k8s)

**Community**: MLOps MENA Community  
**Course**: Kubernetes Course Tasks  
**Repository**: `MLOps k8s` / `Kubernetes-Course-Tasks`  

---

## 📂 Repository Structure

```text
MLOps-k8s/
├── README.md                          # Root repository documentation
├── .gitignore                         # Standard git ignore file
└── Task-1/                            # Session 1: Multi-Container Pods
    ├── README.md                      # Comprehensive guide, verification & diagnosis
    ├── pod.yaml                       # Multi-container Pod (Init + Nginx + Sidecar)
    ├── deployment.yaml                # 3-replica Deployment across nodes
    ├── service.yaml                   # ClusterIP Service on port 80
    ├── ingress.yaml                   # Ingress resource for UI browser access
    ├── pod-failure.yaml               # Diagnostic Pod simulating init failure
    └── failure-patch.yaml             # Deployment failure patch for zero-downtime test
```

---

## 🧪 Completed Tasks

| Task | Topic | Status | Folder |
| :--- | :--- | :---: | :--- |
| **Task 1** | Multi-Container Pods (Init Container & Sidecar with `shareProcessNamespace`) | ✅ Completed | [`Task-1/`](Task-1/README.md) |

---

## 🚀 Quick Start (Task 1)

### 1. Prerequisites
- Running 3-node Kubernetes cluster (`minikube` or `k3d`)
- Ingress controller enabled (`ingress-nginx`)
- `kubectl` configured

### 2. Deploy Application
```bash
# Apply Service and Ingress
kubectl apply -f Task-1/service.yaml
kubectl apply -f Task-1/ingress.yaml

# Apply the Deployment (3 replicas)
kubectl apply -f Task-1/deployment.yaml

# Or apply the single multi-container Pod
kubectl apply -f Task-1/pod.yaml
```

### 3. Access the UI Lab
Open your web browser at:
```
http://localhost/
```
or
```
http://127.0.0.1/
```

You will see:
```text
hello in MLOPS MENA COMMUNITY : <current-timestamp>
```
The timestamp dynamically refreshes every 5 seconds via the `/proc/<pid>/root` sidecar container.

---

For detailed architecture analysis, failure diagnosis (`Init:CrashLoopBackOff`), and zero-downtime rolling update proofs, see [**`Task-1/README.md`**](Task-1/README.md).
