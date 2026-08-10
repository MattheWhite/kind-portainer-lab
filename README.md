# Kubernetes Learning Environment - kind-portainer-lab
A scalable local DevOps laboratory featuring a multi-node **Kind** Kubernetes cluster managed by **Portainer**, demonstrating automated configuration management with ConfigMap-driven deployments, RBAC security, and zero-downtime rolling update strategies.

## 📁 Project Structure

```text
/
├── kind-cluster-config.yaml    # Multi-node Kind cluster definition (1 CP + 2 Workers)
├── docker-compose.yaml         # Portainer CE running via Docker Compose (Local Management)
├── portainer-agent.yaml        # Portainer Agent deployed inside Kind (K8s Manifests)
└── app-stack.yaml              # Sample Nginx App with ConfigMap & Rolling Updates
```

---

## 🎯 Learning Objectives

This environment teaches **core Kubernetes concepts** through hands-on practice:

1.  **Configuration Management**: Decouple configuration from application code using **ConfigMaps**, inject environment variables via `envFrom`, and understand pod restart behaviors.
2.  **Zero-Downtime Deployments**: Master rolling update strategies with `maxSurge: 1` and `maxUnavailable: 0` to ensure availability during updates.
3.  **Multi-Node Cluster Architecture**: Simulate production with a 3-node Kind cluster (1 control-plane + 2 workers) for testing distributed workloads and node affinity.
4.  **Cluster Management & RBAC**: Deploy **Portainer** for visual management, configure ServiceAccounts with ClusterRoleBindings, and understand authorization.
5.  **Hybrid Development Workflows**: Combine Docker Compose for local tooling with Kubernetes for application workloads.

---

## 🚀 Quick Start

### 1. Create the Kind Cluster
Creates a 3-node cluster defined in your config file.
```bash
kind create cluster --config kind-cluster-config.yaml --name learning-cluster
```

### 2. Deploy Portainer CE (Docker Compose)
Starts the Portainer management UI locally.
```bash
docker compose up -d
```
*Access Portainer UI at `https://localhost:9443`*

### 3. Deploy Portainer Agent to Kind
Installs the agent inside the cluster to allow Portainer to manage it. This will create a portainer namespace inside k8s cluster.
```bash
kubectl apply -f portainer-agent-k8s-lb.yaml
```

### 4. Connect Portainer to Kind
1.  Open Portainer UI (`https://localhost:9443`) and create an admin account.
2.  Go to **Environments** → **Add Environment** → **Kubernetes** → **Agent**.
3.  Enter the environment name (e.g., `learning-cluster`).
4.  **Connection Method**:
    *   **Option A (LoadBalancer)**: If you have MetalLB installed, use the External IP of `portainer-agent` on port `9001`.
    *   **Option B (Port Forward - Recommended for Kind)**:
        ```bash
        kubectl port-forward svc/portainer-agent -n portainer 9001:9001
        ```
        Then connect to `localhost:9001` in the Portainer UI.

### 5. Deploy Sample Application
Deploys the Nginx app with ConfigMap injection.
```bash
kubectl apply -f app-stack.yaml
```

---
