"# Kubernetes Fundamentals Lab: Nginx Deployment & Networking

Welcome to the Kubernetes DevOps Learning Path! This repository contains a complete set of manifests designed to teach students how to deploy applications, manage background tasks, and expose services to the outside world.

## 🎯 Learning Objectives
By the end of this lab, students will understand:
1. **Workloads**: How to create a scalable Deployment and a scheduled CronJob.
2. **Networking**: The difference between `ClusterIP`, `NodePort`, and `ExternalName` services.
3. **Traffic Routing**: How to use an Ingress Controller to route external domain traffic.
4. **Job Control**: Managing Job lifecycles using `activeDeadlineSeconds` and `backoffLimit`.

---

## 📂 Project Structure
```text
.
├── sample.dir/
│   ├── jobs/
│   │   └── nginx-job.yaml       # One-time Job / CronJob example
│   └── ...
└── README.md
```

## 🚀 Lab Components

### 1. The Deployment (`nginx-deployment`)
- **Goal**: Ensures 2 replicas of Nginx are always running.
- **Key Concept**: Uses `RollingUpdate` strategy for zero-downtime deployments.
- **Labels**: Uses `tier: prod` and `team: core-banking` for organized resource selection.

### 2. Background Tasks (`nginx-job` & `nginx-cron-job`)
- **Job**: A task that runs to completion.
- **CronJob**: A task that runs on a schedule (e.g., every 10 minutes).
- **Crucial Parameters**:
    - `activeDeadlineSeconds`: Prevents "zombie" jobs by killing them if they run too long.
    - `backoffLimit`: Defines how many times K8s should retry a failing job before giving up.

### 3. Networking Layers (The Service Stack)
We implement three types of services to demonstrate different use cases:
| Service Type | Purpose | Access Level |
| :--- | :--- | :--- |
| **ClusterIP** | Internal communication between pods | Internal Only |
| **NodePort** | Exposes service on a static port on every Node | External (NodeIP:Port) |
| **ExternalName** | Maps a service name to an external DNS (e.g., google.com) | External DNS |

### 4. The Entry Point (Ingress)
- **Goal**: Acts as the HTTP/S load balancer.
- **Flow**: `Client` $\rightarrow$ `Ingress` $\rightarrow$ `Service` $\rightarrow$ `Pod`.

---

## 🛠️ How to Run the Lab

### Prerequisites
- A running Kubernetes cluster (Minikube, Kind, or EKS/GKE).
- `kubectl` installed and configured.
- An Ingress Controller installed (e.g., NGINX Ingress Controller).

### Execution Steps
1. **Deploy the Application**:
   ```bash
   kubectl apply -f deployment.yaml
   ```
2. **Expose the Application**:
   ```bash
   kubectl apply -f service-nodeport.yaml
   ```
3. **Setup Routing**:
   ```bash
   kubectl apply -f ingress.yaml
   ```
4. **Run the Scheduled Task**:
   ```bash
   kubectl apply -f sample.dir/jobs/nginx-cron-job.yaml
   ```

## 📝 Student Challenges
- [ ] Change the `replicas` to 5 and observe the scaling.
- [ ] Modify the `schedule` in the CronJob to run every 5 minutes.
- [ ] Intentionally cause the Job to fail and observe the `backoffLimit` in action.
- [ ] Map a new domain in your `/etc/hosts` file to test the Ingress.
"