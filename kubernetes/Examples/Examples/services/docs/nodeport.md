# Kubernetes Service: NodePort

## What is a NodePort Service?
A **NodePort** service is the simplest way to expose a service to external traffic. It builds upon the ClusterIP service by opening a specific port on **every single Node (Server)** in your cluster.

If you have a cluster with 3 nodes, Kubernetes will open the same port (e.g., 30007) on all 3 nodes. Any traffic sent to any node on that port will be routed to the service, and then to the pods.

---

## Analysis of `node-port.yaml`

### 1. Service Type
```yaml
type: NodePort
```
This tells Kubernetes to not only create an internal IP (ClusterIP) but also to reserve a port on the host machines.

### 2. Port Configuration
There are three different ports here, which often confuses beginners:
- **port: 80**: The internal port of the service (used by other pods inside the cluster).
- **targetPort: 80**: The port the application (Nginx) is actually listening on inside the container.
- **nodePort: 30007**: The port opened on the physical/virtual nodes. This is the port you use in your browser: `http://<Node-IP>:30007`.
  - *Note: The default range for NodePorts is 30000-32767.*

### 3. External Traffic Policy
```yaml
externalTrafficPolicy: Local
```
This is an advanced and important setting for DevOps:
- **Cluster (Default)**: Traffic can be sent to any node, and that node might forward it to a pod on a *different* node. This hides the original client's IP.
- **Local**: Traffic is only sent to pods on the **same node** that received the request. If no pod exists on that node, the packet is dropped. **Benefit:** It preserves the original Client IP address and reduces network hops.

---

## How to Run and Test

### Apply the service:
```bash
kubectl apply -f node-port.yaml
```

### Find your Node IP:
To access the service, you need the IP of one of your cluster nodes:
```bash
kubectl get nodes -o wide
```

### Access the application:
Open your browser and enter:
`http://<ANY_NODE_IP>:30007`