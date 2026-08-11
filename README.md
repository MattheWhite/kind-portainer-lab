# Kubernetes Learning Environment - kind-portainer-lab
A scalable local DevOps laboratory featuring a multi-node **Kind** Kubernetes cluster managed by **Portainer**, demonstrating automated configuration management with ConfigMap-driven deployments, RBAC security, and zero-downtime rolling update strategies.

## 📁 Project Structure

```text
/
├── kind-cluster-config.yaml    # Multi-node Kind cluster definition (1 CP + 2 Workers)
├── docker-compose.yaml         # Portainer CE running via Docker Compose (Local Management)
├── portainer-agent-k8s-lb.yaml # Portainer Agent deployed inside Kind (K8s Manifests)
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
kind create cluster --config kind-cluster-config.yaml --name lab-cluster
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

> **⚠️ Expect `EXTERNAL-IP: <pending>` — this is normal, not a failure.**
> `kubectl get svc -n portainer` will show `portainer-agent  LoadBalancer  10.96.x.x  <pending>  9001:3xxxx/TCP` forever.
> Kind ships **no load-balancer controller**, so a `type: LoadBalancer` Service never gets an external IP.
> Two separate problems have to be solved to connect:
>
> 1. **No external IP** → use the **NodePort** the Service already allocated (the `3xxxx` half of `9001:3xxxx/TCP`).
> 2. **Network isolation** → Portainer is a *container* on the `portainer_network` bridge; the Kind nodes are
>    containers on the `kind` bridge. From inside Portainer, `localhost` is Portainer itself, and the
>    `10.96.x.x` ClusterIP does not exist (ClusterIPs are only routable *inside* the cluster).

#### 4a. Verify the agent is actually up

```bash
kubectl get pods -n portainer                 # portainer-agent-xxxx must be 1/1 Running
kubectl get svc portainer-agent -n portainer -o jsonpath='{.spec.ports[0].nodePort}'
```

Note the NodePort — it is assigned randomly at apply time and changes every time the manifest is re-applied
to a fresh cluster. Every Kind node listens on it (kube-proxy programmes it cluster-wide).

#### 4b. Join Portainer to the Kind Docker network

```bash
docker network ls                             # confirm the Kind network is named "kind"
docker network connect kind portainer         # "portainer" = container_name from docker-compose.yaml
docker ps --filter name=lab-cluster           # e.g. lab-cluster-control-plane
```

A container can be attached to multiple Docker networks at once, so Portainer keeps its published
`9443`/`8000` ports on `portainer_network` **and** gains a second interface on `kind`.

#### 4c. Register the environment

1.  Open Portainer UI (`https://localhost:9443`) and create an admin account.
2.  Go to **Environments** → **Add Environment** → **Kubernetes** → **Agent**.
3.  **Name**: `lab-cluster`
4.  **Environment URL**: `lab-cluster-control-plane:<nodePort>` — for example `lab-cluster-control-plane:32492`

    No scheme (`https://`) and no trailing path. Docker's embedded DNS on the user-defined `kind` network
    resolves the node container name, so nothing has to be hardcoded to an IP that changes on every
    `kind delete cluster` / `kind create cluster` cycle.

<details>
<summary><b>Fallback: <code>kubectl port-forward</code></b> (only if you don't want to touch Docker networks)</summary>

```bash
kubectl port-forward -n portainer svc/portainer-agent --address 0.0.0.0 9001:9001
```

Then use `host.docker.internal:9001` as the Environment URL — **not** `localhost:9001`, which from inside the
Portainer container points at Portainer itself. `--address 0.0.0.0` is required because the default bind of
`127.0.0.1` is unreachable from a container.

Trade-offs: the tunnel dies when the terminal closes, and it targets a single pod directly rather than routing
through the Service — so it hides exactly the networking behaviour this lab is meant to teach.
</details>

<details>
<summary><b>Advanced: real LoadBalancer IPs with <code>cloud-provider-kind</code></b></summary>

`cloud-provider-kind` is the current upstream way to get working `type: LoadBalancer` Services in Kind
(it superseded MetalLB for this use case). It runs as a container on the host, watches for LB Services, and
assigns IPs on the Docker network:

```bash
docker run -d --name cloud-provider-kind --network kind \
  -v /var/run/docker.sock:/var/run/docker.sock \
  registry.k8s.io/cloud-provider-kind/cloud-controller-manager:v0.7.0
```

`EXTERNAL-IP` then resolves to a Docker-network address. It is reachable from other containers on `kind`
(including Portainer, once joined per 4b) but **not** from the Windows host directly — so this complements
step 4b rather than replacing it.
</details>

### 5. Deploy Sample Application
Deploys the Nginx app with ConfigMap injection.
```bash
kubectl apply -f app-stack.yaml
```

---

