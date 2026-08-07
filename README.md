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
