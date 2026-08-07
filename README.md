# Kubernetes Learning Environment - kind-portainer-lab
A scalable local DevOps laboratory featuring a multi-node Kind Kubernetes cluster managed by Portainer, demonstrating automated configuration management with ConfigMap-driven deployments, RBAC security, and zero-downtime rolling update strategies.

## 📁 Project Structure

```text
learning/
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
