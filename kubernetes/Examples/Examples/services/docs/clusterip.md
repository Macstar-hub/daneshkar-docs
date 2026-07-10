# Kubernetes Service: ClusterIP

## What is a ClusterIP Service?
In Kubernetes, a **ClusterIP** is the default type of Service. It provides a stable IP address that is only accessible from **inside** the Kubernetes cluster. 

Think of it as an "Internal Load Balancer." If you have multiple pods running your application, ClusterIP allows other services in the cluster to talk to them using a single name or IP, without needing to know the individual IP addresses of the pods (which can change if a pod restarts).

---

## Analysis of `clusterip.yaml`

Here is the breakdown of the example provided:

### 1. Metadata
- **Name**: `nginx-service` (The name used to identify this service).
- **Namespace**: `daneshkar` (The logical partition where this service lives).
- **Labels**: Used for organizing and grouping resources (e.g., `team: core-banking`).

### 2. The Selector (The "Bridge")
```yaml
selector:
  app: nginx
  tier: prod
  team: core-banking
```
The **selector** is the most important part. It tells Kubernetes: *"Send traffic to any Pod that has these exact labels."* If a pod has `app: nginx`, `tier: prod`, and `team: core-banking`, it will receive traffic from this service.

### 3. Ports
- **port: 80**: This is the port that the service exposes **inside the cluster**. Other pods will call this service on port 80.
- **targetPort: 80**: This is the port the **application (the pod)** is actually listening on.

### 4. Session Affinity
- **sessionAffinity: ClientIP**: This ensures that requests from a specific client IP are always routed to the same pod for the duration of the session (Sticky Sessions). The default is usually `None`.

---

## DNS and Connectivity
Because this is a ClusterIP service, Kubernetes creates a DNS entry for it. Based on your file, the Full Qualified Domain Name (FQDN) is:
`nginx-service.daneshkar.svc.cluster.local`

Any other pod in the cluster can simply send a request to `http://nginx-service` (if in the same namespace) to reach the Nginx pods.

## How to Run and Test

### Apply the service:
```bash
kubectl apply -f clusterip.yaml
```

### Verify the service:
```bash
kubectl get svc -n daneshkar
```

### Test from inside the cluster:
Since ClusterIP is not accessible from your browser outside the cluster, you must run a temporary pod to test it:
```bash
kubectl run curl-test --image=radial/busyboxplus:curl -i --tty --rm
# Inside the pod, run:
curl http://nginx-service.daneshkar
```