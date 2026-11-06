# Task Session 1: Multi-Container Pods (Init Container & Sidecar)

**Course**: MLOps MENA Community - Kubernetes Course  
**Task Repository Reference**: `MLOpsMENACommunity/Kubernetes-Course-Tasks`  
**Folder**: `Task-1/`  
**Target Cluster**: 3-Node Kubernetes Cluster (`k3d-multinodes` / `minikube`)  

---

## 📌 Table of Contents
1. [Overview & Requirements](#-overview--requirements)
2. [Architecture & Design](#-architecture--design)
   - [Why `shareProcessNamespace: true`?](#why-shareprocessnamespace-true)
   - [The `/proc/<pid>/root` Mechanism](#the-procpidroot-mechanism)
3. [Container Breakdown](#-container-breakdown)
4. [Manifests Reference](#-manifests-reference)
5. [Step-by-Step Deployment Guide](#-step-by-step-deployment-guide)
   - [1. Deploying the Multi-Container Pod](#1-deploying-the-multi-container-pod)
   - [2. Deploying the Multi-Node Deployment, Service & Ingress](#2-deploying-the-multi-node-deployment-service--ingress)
6. [Verification & Dynamic Timestamp Updates](#-verification--dynamic-timestamp-updates)
   - [Local Browser UI Lab](#local-browser-ui-lab)
   - [Live CLI / Curl Verification](#live-cli--curl-verification)
7. [Failure Simulation & Kubernetes Lifecycle Analysis](#-failure-simulation--kubernetes-lifecycle-analysis)
   - [1. Pod-Level Init Failure (Init:CrashLoopBackOff)](#1-pod-level-init-failure-initcrashloopbackoff)
   - [2. Deployment Rolling Update & Zero Downtime Proof](#2-deployment-rolling-update--zero-downtime-proof)
   - [3. Recovery & Rollback](#3-recovery--rollback)
8. [Summary & Key Learnings](#-summary--key-learnings)

---

## 📝 Overview & Requirements

Deploy an Nginx "Hello" page on a 3-node Kubernetes cluster that displays the live timestamp in the exact format:
```text
hello in MLOPS MENA COMMUNITY : 2026-10-15 10:15:05
```

### Constraints & Architecture Rules:
- **No Shared Volumes**: Neither `emptyDir` nor `PersistentVolumeClaim` is used.
- **Process Namespace Sharing**: Uses `shareProcessNamespace: true` in the Pod spec.
- **Inter-Container Communication**: Sidecar writes directly to Nginx's file tree via Linux `/proc/<pid>/root`.
- **Three Containers per Pod**:
  1. `init-done` (Init Container)
  2. `nginx` (Main Web Server)
  3. `time-updater` (Sidecar Dynamic Updater)
- **Networking**: Kubernetes `Service` and `Ingress` exposing the application on port `80` (`http://localhost` / `http://127.0.0.1`).
- **Failure Lifecycle**: Simulate an init container failure (`exit 1`), inspect `Init:CrashLoopBackOff`, prove zero downtime under a rolling update, and restore service.

---

## 🏛 Architecture & Design

```mermaid
flowchart TD
    subgraph Pod["Kubernetes Pod (shareProcessNamespace: true)"]
        subgraph InitPhase["Phase 1: Init Container"]
            Init["init-done (busybox:1.36)\necho done && exit 0"]
        end

        subgraph AppPhase["Phase 2: Running App Containers"]
            Main["nginx (nginx:alpine)\nPID X in shared namespace\nServes /usr/share/nginx/html/index.html\nPort 80"]
            Sidecar["time-updater (busybox:1.36)\nFinds NGINX_PID via pgrep\nWrites every 5s to /proc/NGINX_PID/root/.../index.html"]
        end
    end

    InitPhase -->|Success (Exit 0)| AppPhase
    Sidecar -.->|Direct /proc filesystem write| Main

    Client["User Browser / Curl"] --> Ingress["Nginx Ingress (port 80)"]
    Ingress --> Service["mlops-nginx-service (ClusterIP: 80)"]
    Service --> Main
```

### Why `shareProcessNamespace: true`?
By default, Docker and Kubernetes isolate each container in its own PID namespace (`CLONE_NEWPID`). When `shareProcessNamespace: true` is configured:
1. All containers within the same Pod share a single PID namespace.
2. The processes of one container are visible to all other containers in the Pod via standard process listings (`ps`, `pgrep`).
3. PID 1 is the Pod sandbox pause process, preventing container termination when an internal process stops.

### The `/proc/<pid>/root` Mechanism
In Linux, `/proc/<pid>/root` is a virtual symlink pointing to the root filesystem (`/`) of the process with PID `<pid>`.
Because both containers run as `root` (UID 0) within the same PID and user namespaces:
- The `time-updater` sidecar discovers Nginx's PID:
  ```sh
  PID=$(pgrep -o nginx)
  ```
- The sidecar updates `index.html` directly at:
  ```sh
  /proc/$PID/root/usr/share/nginx/html/index.html
  ```
- Nginx immediately serves the updated file to incoming HTTP clients without requiring any volume mount.

---

## 📦 Container Breakdown

| Container | Role | Image | Command / Logic |
| :--- | :--- | :--- | :--- |
| **`init-done`** | **Init Container** | `busybox:1.36` | `echo done && exit 0`<br>Ensures prerequisites are met before app starts. |
| **`nginx`** | **Main Web Server** | `nginx:alpine` | Writes initial `index.html` with timestamp and executes `nginx -g 'daemon off;'` on port 80. |
| **`time-updater`** | **Sidecar Container**| `busybox:1.36` | Loops every 5 seconds, locates Nginx PID, and overwrites `/proc/$PID/root/.../index.html` with current timestamp. |

---

## 📁 Manifests Reference

All manifests are stored in this directory:
- [`pod.yaml`](pod.yaml) — Standalone 3-container Pod with `shareProcessNamespace: true`.
- [`deployment.yaml`](deployment.yaml) — Production-grade 3-replica Deployment distributed across the 3 worker nodes.
- [`service.yaml`](service.yaml) — ClusterIP Service exposing port 80.
- [`ingress.yaml`](ingress.yaml) — Ingress resource routing port 80 traffic to the Service.
- [`pod-failure.yaml`](pod-failure.yaml) — Diagnostic Pod with forced init failure (`exit 1`) for lifecycle analysis.
- [`failure-patch.yaml`](failure-patch.yaml) — Patch to demonstrate rolling update zero-downtime and crash diagnosis.

---

## 🚀 Step-by-Step Deployment Guide

### 1. Deploying the Multi-Container Pod
To run the standalone multi-container Pod:
```bash
kubectl apply -f Task-1/pod.yaml
```
Verify Pod status:
```bash
kubectl get pod mlops-multi-container-pod -o wide
```

Output:
```text
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE                     NOMINATED NODE   READINESS GATES
mlops-multi-container-pod   2/2     Running   0          97s   10.42.1.60   k3d-multinodes-agent-1   <none>           <none>
```

Inspect the `init-done` init container output:
```bash
kubectl logs mlops-multi-container-pod -c init-done
```
Output:
```text
done
```

---

### 2. Deploying the Multi-Node Deployment, Service & Ingress
To deploy the full highly-available stack with 3 replicas across the 3-node cluster:
```bash
kubectl apply -f Task-1/deployment.yaml
kubectl apply -f Task-1/service.yaml
kubectl apply -f Task-1/ingress.yaml
```

Check the 3 replicas scheduled across all cluster nodes:
```bash
kubectl get pods -l app=mlops-nginx -o wide
```
Output:
```text
NAME                                      READY   STATUS    RESTARTS   AGE    IP            NODE                      NOMINATED NODE   READINESS GATES
mlops-nginx-deployment-5f8d8767d8-527ms   2/2     Running   0          5m     10.42.0.191   k3d-multinodes-server-0   <none>           <none>
mlops-nginx-deployment-5f8d8767d8-kfn4x   2/2     Running   0          5m     10.42.3.30    k3d-multinodes-agent-2    <none>           <none>
mlops-nginx-deployment-5f8d8767d8-lkb8d   2/2     Running   0          5m     10.42.2.115   k3d-multinodes-agent-0    <none>           <none>
```

Check Service and Ingress:
```bash
kubectl get svc mlops-nginx-service
kubectl get ingress mlops-nginx-ingress
```

---

## 🧪 Verification & Dynamic Timestamp Updates

### Local Browser UI Lab
Open any web browser and navigate to:
```
http://127.0.0.1/
```
or
```
http://localhost/
```

The web page displays:
```
hello in MLOPS MENA COMMUNITY : 2026-11-06 18:30:38
```
Refresh the page after 5 seconds to observe the sidecar updating the timestamp in real-time.

---

### Live CLI / Curl Verification
Verify dynamic updates directly from terminal:
```powershell
1..3 | ForEach-Object { 
    curl.exe -s -4 http://127.0.0.1:80
    Start-Sleep -Seconds 5 
}
```

Terminal Output:
```text
hello in MLOPS MENA COMMUNITY : 2026-11-06 16:30:38
hello in MLOPS MENA COMMUNITY : 2026-11-06 16:30:43
hello in MLOPS MENA COMMUNITY : 2026-11-06 16:30:48
```
Notice how every 5 seconds, the sidecar writes the updated timestamp into `/proc/<pid>/root/usr/share/nginx/html/index.html` with zero persistent volumes.

---

## 💥 Failure Simulation & Kubernetes Lifecycle Analysis

### 1. Pod-Level Init Failure (`Init:CrashLoopBackOff`)
Apply the intentional failure manifest where `init-done` exits with code `1`:
```bash
kubectl apply -f Task-1/pod-failure.yaml
```

Observe the Pod lifecycle:
```bash
kubectl get pod mlops-failing-pod
```
Output:
```text
NAME                READY   STATUS       RESTARTS      AGE
mlops-failing-pod   0/2     Init:Error   2 (28s ago)   30s
```

After backoff threshold:
```bash
kubectl get pod mlops-failing-pod
```
Output:
```text
NAME                READY   STATUS                  RESTARTS      AGE
mlops-failing-pod   0/2     Init:CrashLoopBackOff   3 (35s ago)   52s
```

#### Diagnosis via `kubectl logs` and `kubectl describe`:
```bash
kubectl logs mlops-failing-pod -c init-done
```
Output:
```text
Simulating init failure...
```

Inspect events:
```bash
kubectl describe pod mlops-failing-pod
```
Output:
```text
Init Containers:
  init-done:
    State:          Terminated
      Reason:       Error
      Exit Code:    1
Containers:
  nginx:
    State:          Waiting
      Reason:       PodInitializing
  time-updater:
    State:          Waiting
      Reason:       PodInitializing
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  60s                default-scheduler  Successfully assigned default/mlops-failing-pod to k3d-multinodes-server-0
  Normal   Started    19s (x4 over 60s)  kubelet            Container started
  Warning  BackOff    18s (x3 over 58s)  kubelet            Back-off restarting failed container init-done
```

> **Key Kubernetes Lifecycle Behavior**:
> Kubernetes enforces strict sequential initialization: regular containers (`nginx` and `time-updater`) remain in `Waiting: PodInitializing` and are **never executed** as long as `init-done` does not exit with code `0`.

---

### 2. Deployment Rolling Update & Zero Downtime Proof

Simulate an accidental buggy release where the init container fails:
```bash
kubectl patch deployment mlops-nginx-deployment --patch-file Task-1/failure-patch.yaml
```

Monitor Pods:
```bash
kubectl get pods -l app=mlops-nginx
```
Output:
```text
NAME                                      READY   STATUS       RESTARTS      AGE
mlops-nginx-deployment-5f8d8767d8-527ms   2/2     Running      0             4m19s
mlops-nginx-deployment-5f8d8767d8-kfn4x   2/2     Running      0             4m19s
mlops-nginx-deployment-5f8d8767d8-lkb8d   2/2     Running      0             4m18s
mlops-nginx-deployment-6c6987b576-46vf7   0/2     Init:Error   2 (14s ago)   19s
```

Test live traffic while the new pod is crashing:
```bash
curl.exe -s -4 http://127.0.0.1:80
```
Output:
```text
hello in MLOPS MENA COMMUNITY : 2026-11-06 16:26:02
```

> **Zero Downtime Guarantee**:
> Because Kubernetes rolling updates wait for newly created Pods to pass readiness and initialization checks before terminating old Pods, the three healthy v1 Pods continue serving 100% of traffic without a single dropped packet.

---

### 3. Recovery & Rollback
Rollback the failed deployment to the last known good revision:
```bash
kubectl rollout undo deployment/mlops-nginx-deployment
```
Output:
```text
deployment.apps/mlops-nginx-deployment rolled back
```

Verify all replicas are healthy:
```bash
kubectl get pods -l app=mlops-nginx
```
Output:
```text
NAME                                      READY   STATUS    RESTARTS   AGE
mlops-nginx-deployment-5f8d8767d8-527ms   2/2     Running   0          5m14s
mlops-nginx-deployment-5f8d8767d8-kfn4x   2/2     Running   0          5m14s
mlops-nginx-deployment-5f8d8767d8-lkb8d   2/2     Running   0          5m13s
```

The failing pod is cleanly terminated and removed from the cluster.

---

## 🎯 Summary & Key Learnings

1. **Process Namespace Sharing (`shareProcessNamespace: true`)**: Enabled inter-container file access via `/proc/<pid>/root` without provisioning shared storage volumes or PVCs.
2. **Init Container Guarantees**: Kubernetes guarantees that application containers only launch upon successful completion (`exit 0`) of all init containers.
3. **Resilience & Zero Downtime**: Kubernetes Deployment controllers isolate initialization failures within canary/new pods, shielding end-users from outages during bad deployments.
4. **Ingress & Service Integration**: Connected external browser traffic seamlessly via Ingress controller to cluster services.
