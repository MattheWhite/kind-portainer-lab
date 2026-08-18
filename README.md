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
6.  **Network Boundaries**: Understand why a Service that "works" is still unreachable from your browser, and which mechanism crosses each layer.

---

## 🗺️ Architecture: The Four Network Layers

Almost every "why can't I reach it?" problem in this lab comes down to one idea:
**there are four networks stacked on top of each other, and every boundary needs an explicit crossing mechanism.**

```text
┌─ Windows host ──────────────────────────────────────────────────┐
│  browser → localhost:9443       localhost:8080                  │
│                  │                    │                         │
│             [published port]   [port-forward tunnel]            │
│                  │                    │                         │
│  ┌─ Docker ──────┼────────────────────┼────────────────────────┐│
│  │  network: portainer_network   network: kind                 ││
│  │  ┌──────────────┐            ┌───────────────────────────┐  ││
│  │  │  portainer   │──joined────│ lab-cluster-control-plane │  ││
│  │  │  (container) │  (4b)      │ lab-cluster-worker        │  ││
│  │  └──────────────┘            │ lab-cluster-worker2       │  ││
│  │                              │   172.18.x.x              │  ││
│  │       ┌──────────────────────┴──────────────────────┐    │  ││
│  │       │  Kubernetes                                 │    │  ││
│  │       │  Services  10.96.x.x    (ClusterIP range)   │    │  ││
│  │       │  Pods      10.244.x.x   (Pod CIDR)          │    │  ││
│  │       │  ├─ ns portainer : portainer-agent   :9001  │    │  ││
│  │       │  └─ ns default   : backend pods      :80    │    │  ││
│  │       └─────────────────────────────────────────────┘    │  ││
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

| Layer | Address space | How to cross into it |
|-------|---------------|----------------------|
| Pod network | `10.244.x.x` | in-cluster only; never address a pod IP directly |
| Service network | `10.96.x.x` (ClusterIP) | in-cluster only — DNS name + port |
| Node / Docker network | `172.18.x.x`, container names | `NodePort`; reachable from any container on the `kind` network |
| Windows host | `localhost` | `kubectl port-forward`, or Kind `extraPortMappings` |

**Key consequences**

*   A `type: ClusterIP` Service is *by design* invisible from Windows. It is not broken.
*   A `type: LoadBalancer` Service in Kind stays `<pending>` forever — there is no cloud LB to provision.
*   `localhost` means something different in every box above. Inside the Portainer container, `localhost` is Portainer.
*   `LoadBalancer` ⊃ `NodePort` ⊃ `ClusterIP`. A LoadBalancer Service **already allocates a NodePort**, which is why
    step 4 works without changing the manifest.

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

Verify the objects were created and the pods were scheduled across both workers:

```bash
kubectl get pods -o wide            # 2/2 Running, on different nodes
kubectl get svc backend-internal-svc
kubectl get configmap app-config -o yaml
```

Confirm the ConfigMap actually reached the container as environment variables:

```bash
kubectl exec deploy/backend-service -- env | grep -E 'APP_ENV|LOG_LEVEL'
# APP_ENV=local-onprem
# LOG_LEVEL=debug
```

> **Note:** `nginx:alpine` does not *use* these variables — they are injected to demonstrate the mechanism, not to
> configure the web server. The lesson is the injection itself, and the restart behaviour described in step 7.

### 6. Reaching the Application

`backend-internal-svc` is `type: ClusterIP`. It is reachable **only from inside the cluster**, so
`http://localhost:8080` in your browser will fail — correctly, and by design. First prove the Service works
from where it is meant to work:

```bash
kubectl run tmp --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s http://backend-internal-svc:8080
```

That should print the nginx welcome HTML. Two things just happened: CoreDNS resolved the *Service name* to a
ClusterIP, and kube-proxy load-balanced the request to one of the two backend pods.

Then pick one of the three ways out, in increasing order of production-realism:

| Mechanism | Reaches | Persistence | Cost |
|-----------|---------|-------------|------|
| `kubectl port-forward` | your laptop only | dies with the terminal | none — no manifest change |
| `NodePort` + `extraPortMappings` | your laptop, persistently | survives restarts | requires cluster recreate |
| Ingress controller | many apps behind one entry point | survives restarts | install ingress-nginx |

#### 6a. Option A — `port-forward` (quickest, no changes)

```bash
kubectl port-forward svc/backend-internal-svc 8080:8080
```

Open `http://localhost:8080`. The left number is the **Windows** port (choose freely), the right number is the
**Service** port. This tunnels through the API server, so it needs no networking changes at all. It is the correct
tool for poking at a ClusterIP Service — but it bypasses the normal data path, so it teaches you the least.

#### 6b. Option B — NodePort + Kind `extraPortMappings` (persistent)

Kind can publish a node-container port to the Windows host, but **only at cluster creation time**. Add the mapping
to `kind-cluster-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080     # NodePort inside the node container
    hostPort: 8080           # port on Windows
    protocol: TCP
- role: worker
- role: worker
```

Then pin the Service to that NodePort in `app-stack.yaml`:

```yaml
spec:
  type: NodePort
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 80
    nodePort: 30080          # must be in the 30000-32767 range
```

Mapping on the control-plane node is fine even though the pods run on the workers: kube-proxy programmes the
NodePort on **every** node and forwards to wherever the pods actually are.

> **⚠️ This requires `kind delete cluster --name lab-cluster` and a recreate, which destroys everything.**
> You will need to re-run step 3 (agent) and re-register the environment in Portainer with its **new** randomly
> assigned NodePort.

#### 6c. Option C — Ingress controller (production-shaped)

One entry point, routing many apps by hostname or path. Kind's documented setup needs an `ingress-ready` node
label plus port mappings for 80/443, again at creation time:

```yaml
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
```

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=120s
```

*(Pin a release tag instead of `main` if you want a reproducible lab.)* The Service can then stay `ClusterIP`,
with an `Ingress` object pointing at it — which is exactly how real clusters expose HTTP workloads.

### 7. Exercise the Learning Objectives

**ConfigMap changes do not restart pods.** Edit the ConfigMap, then re-check the pod's environment:

```bash
kubectl edit configmap app-config          # change LOG_LEVEL to "info"
kubectl exec deploy/backend-service -- env | grep LOG_LEVEL   # still "debug"!
```

Environment variables are fixed at container start, so the running pods never see the new value. This is the
single most common ConfigMap gotcha. Force a new generation of pods:

```bash
kubectl rollout restart deploy/backend-service
kubectl exec deploy/backend-service -- env | grep LOG_LEVEL   # now "info"
```

*(Mounting the ConfigMap as a **volume** instead would update the file in-place without a restart — a useful
follow-up experiment.)*

**Watch the zero-downtime rollout.** In one terminal:

```bash
kubectl get pods -w
```

In another:

```bash
kubectl rollout restart deploy/backend-service
kubectl rollout status deploy/backend-service
```

With `maxSurge: 1` and `maxUnavailable: 0`, Kubernetes creates one *extra* pod and waits for it to become Ready
**before** terminating an old one — the replica count never drops below 2. Change `maxUnavailable` to `1` and
repeat to see the difference.

**Inspect the object hierarchy.** A Deployment does not create pods directly:

```bash
kubectl get deploy,rs,pods -l app=backend
kubectl rollout history deploy/backend-service
kubectl rollout undo deploy/backend-service      # roll back to the previous ReplicaSet
```

`Deployment → ReplicaSet → Pod`. Each rollout creates a new ReplicaSet, and the old ones are retained precisely
so that `rollout undo` can scale them back up.

---

## 📋 Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Docker Desktop / Engine | 20.10+ | Container runtime |
| Kind | 0.20+ | Local Kubernetes cluster |
| kubectl | 1.28+ | Kubernetes CLI |
| Docker Compose | 2.0+ | Portainer CE deployment |

**Resource Requirements**: Minimum 8 GB RAM, 4 CPU cores recommended for a smooth multi-node experience.

---

## 📄 File Descriptions

### `kind-cluster-config.yaml`
Defines the **Kind cluster topology**. "Kind" means *Kubernetes IN Docker*: every "node" is really a Docker
container running a kubelet and a container runtime.
*   1 control-plane node — API server, scheduler, controller-manager, etcd.
*   2 worker nodes — workload execution only.
*   All three node containers sit on a Docker bridge network named `kind`.
*   Enables testing of pod distribution across nodes.

> **This is the only file you cannot change without destroying the cluster.** Node counts,
> `extraPortMappings` and `kubeadmConfigPatches` are all read once, at `kind create cluster` time.

### `docker-compose.yaml`
Runs **Portainer CE** as a standalone Docker container, deliberately **outside** the Kubernetes cluster — it is
the management console, not a workload.
*   Mounts the Docker socket (`/var/run/docker.sock`) so it can also manage plain Docker containers.
*   Publishes port `9443` (HTTPS UI) and `8000` (Edge Agents) to the Windows host.
*   Uses a named volume (`portainer_data`) to persist settings across recreates.
*   Attaches to its own network, `portainer_network` — which is why step 4b has to bridge it onto `kind`.
    A container can hold interfaces on several Docker networks at once, so joining `kind` does not disturb
    the published `9443`/`8000` ports.

This split — Compose for local tooling, Kubernetes for workloads — is the "hybrid workflow" of objective 5.

### `portainer-agent-k8s-lb.yaml`
Deploys the **Portainer Agent** inside the Kubernetes cluster. Four objects working together:

| Object | Role |
|--------|------|
| `Namespace: portainer` | Logical partition for names and RBAC scope |
| `ServiceAccount: portainer-sa-clusteradmin` | The agent pod's **identity** when it calls the API server |
| `ClusterRoleBinding → cluster-admin` | Grants that identity full cluster permissions |
| `Service: portainer-agent` (LoadBalancer) | Entry point on port 9001 — stays `<pending>` in Kind |
| `Service: portainer-agent-headless` | `clusterIP: None`; lets multiple agent replicas discover each other via `AGENT_CLUSTER_ADDR` |
| `Deployment: portainer-agent` | The agent itself, `portainer/agent:2.42.0` |

**The RBAC lesson:** pods authenticate to the API server *as a ServiceAccount*, and a `RoleBinding` /
`ClusterRoleBinding` is what turns that identity into permissions. `cluster-admin` here is deliberately maximal
because Portainer needs to manage everything; in production you would scope it down to the narrowest role that
still works.

### `app-stack.yaml`
Demonstrates **application deployment patterns**:
*   **ConfigMap `app-config`**: stores `APP_ENV` and `LOG_LEVEL` in etcd, decoupled from the container image.
*   **Deployment `backend-service`**: declares desired state ("2 replicas of `nginx:alpine`"). It does **not**
    create pods directly — it creates a **ReplicaSet**, which creates the pods.
*   **`envFrom.configMapRef`**: loads *all* ConfigMap keys as environment variables. Fixed at container start
    (see step 7).
*   **RollingUpdate `maxSurge: 1` / `maxUnavailable: 0`**: add at most one extra pod, never drop below 2 healthy —
    the new pod must be Ready before an old one is terminated.
*   **Service `backend-internal-svc`**: a stable virtual IP and DNS name in front of pods whose IPs churn on every
    restart. `port: 8080` is what clients call; `targetPort: 80` is where nginx actually listens. That indirection
    is intentional — it lets you change the container port without touching any client.

---

## 🔧 Troubleshooting Quick Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| `EXTERNAL-IP` stuck at `<pending>` | Kind has no load-balancer controller | Use the auto-allocated NodePort (step 4a), or install `cloud-provider-kind` |
| Portainer can't reach `localhost:9001` | Inside the container, `localhost` **is** Portainer | Join the `kind` network and use `lab-cluster-control-plane:<nodePort>` (step 4b) |
| Portainer can't reach `10.96.x.x` | ClusterIPs are routable only inside the cluster | Same as above |
| `http://localhost:8080` refuses to connect | Service is `type: ClusterIP` — no host exposure exists | `port-forward`, or NodePort + `extraPortMappings` (step 6) |
| `port-forward` works locally but not from a container | Default bind is `127.0.0.1` | Add `--address 0.0.0.0` and use `host.docker.internal` |
| ConfigMap edited but pod env unchanged | Env vars are frozen at container start | `kubectl rollout restart deploy/backend-service` |
| Environment vanished after recreating the cluster | New cluster = new NodePort and new certs | Re-run step 3, re-register with the new NodePort |

### Teardown

```bash
kubectl delete -f app-stack.yaml
kubectl delete -f portainer-agent-k8s-lb.yaml
kind delete cluster --name lab-cluster
docker compose down            # add -v to also drop the portainer_data volume
```

---
