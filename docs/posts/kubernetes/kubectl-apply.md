---
title: "Understanding `kubectl apply` in Kubernetes"
date: 2025-04-22
tags:
  - Kubernetes
  - kubectl
---

Kubernetes is often seen as a world of Pods, but it's just as much a world of controllers. To understand how everything works in a Kubernetes environment, the key question to ask is:

> **What happens when we apply a manifest via `kubectl` to a cluster?**

This is similar in spirit to the classic interview question:

> **What happens when you type a URL into your browser?**

---
### Starting Point: `kubectl apply`

When we apply a manifest using `kubectl`, we are effectively sending an HTTPS request to the Kubernetes API server (`kube-apiserver`). Depending on whether the resource exists, the request could use `POST`, `PUT`, or `PATCH`. The `kubectl` client handles this intelligently using a **3-way strategic merge**:

- It compares the previous configuration (last-applied), the current cluster state, and the new manifest.
- The last-applied configuration is stored in an annotation on the resource: `kubectl.kubernetes.io/last-applied-configuration`.
- This allows for non-destructive updates and idempotent operations.

This makes `kubectl apply` different from `create` or `replace`, which respectively fail if a resource exists or overwrite everything.

You could simulate part of this interaction using `curl`:

```bash
curl -k -H "Authorization: Bearer <TOKEN>" \
     -H "Content-Type: application/yaml" \
     --data-binary @manifest.yaml \
     https://<K8S-API>/api/v1/namespaces/<namespace>/pods
```

This command only works for **creating** resources and requires the correct API path:

- Deployments: `/apis/apps/v1/namespaces/default/deployments`
- Services: `/api/v1/namespaces/default/services`

Once validated, the `kubectl` client sends an HTTPS request from your machine to the `kube-apiserver`, a component of the Kubernetes control plane.

> **Note**: Manifests can be written in `yaml` or `json`, both of which are understood by `kubectl`.

>**Note**: If you directly edit a resource using `kubectl edit` or `kubectl patch` (not `apply`), the `last-applied-configuration` will be **outdated**, and future `kubectl apply` operations might behave unexpectedly.
>**That’s why `kubectl apply` should be your primary method for managing resources in declarative workflows.**

---

### Kubernetes: A System of APIs and a Key-Value Store

The `kube-apiserver` serves as the frontend to the control plane and handles all REST API requests. It performs several steps:

- **Authentication**: Confirms that the requester is valid (via RBAC, service accounts, etc.).
- **Authorization**: Checks if the requester is allowed to perform the action.
- **Validation**: Verifies that the request complies with the Kubernetes schema.
- **Admission Control**: Applies policies (e.g., quotas, security checks).

If all checks pass, the `kube-apiserver` persists the resource in `etcd`, the distributed key-value store that acts as Kubernetes' source of truth.

Kubernetes operates on a **declarative model**, meaning you declare the desired state, and the system works to maintain it:

- For **reads**: the API server fetches data from `etcd`.
- For **writes**: the API server stores new or updated objects in `etcd`.

---
### The 3-Way Reconciliation Loop: API Server, etcd, Controllers

At the heart of Kubernetes lies its reconciliation loop, which acts like a feedback-driven 3-way handshake between:

1. **You (Desired State)** — you declare what you want via `kubectl apply`.
2. **etcd (Current State)** — it stores the live state of all objects.
3. **Controller (Actual State Manager)** — it watches the state and enforces convergence.    

The flow looks like this:

- You apply a Deployment (desired state).
- The `kube-apiserver` stores it in `etcd`.
- The **Deployment Controller**, running inside the `kube-controller-manager`, watches for changes to Deployments.
- It compares desired vs actual, and if there's a difference (e.g., fewer replicas than desired), it takes action (e.g., create new ReplicaSets or Pods).    
- These actions again go through the API server and are saved into `etcd`.


This loop continues until the actual state matches the desired state. This is the magic behind Kubernetes self-healing.

---
### Who Watches Over Kubernetes Objects?

Every object in Kubernetes is watched by a controller that ensures it matches the desired state:

- **Pods** are managed by a **ReplicaSet**.
- **ReplicaSets** are managed by a **Deployment**.
- These are managed by controllers running in the **kube-controller-manager** binary.

So, when a Deployment is applied, the **Deployment Controller** observes the Deployment resource and ensures the defined number of Pods are running. If there's a mismatch between the current and desired state, the controller adjusts accordingly (e.g., scaling Pods, initiating rolling updates).

> **Note**: A **Kubernetes Operator** is a custom controller that extends Kubernetes to manage complex applications by packaging domain-specific logic.

---
### Where Does a Pod Live? The Scheduler Decides

Once the Deployment Controller determines that new Pods are needed, it delegates the scheduling decision to the **Scheduler**. The Scheduler evaluates available Nodes using:

- Resource availability (CPU, memory)
- Taints and tolerations
- Affinity/anti-affinity rules
- Node selectors and constraints

It then sets the `nodeName` field in the Pod spec to assign it to a specific Node.

> **Note**: The Scheduler only makes placement decisions; it doesn’t run Pods.

---
### When the Pod Is Born

Once scheduled, the Pod becomes part of the desired state for the **Kubelet**, which runs on each Node. The Kubelet:

- Pulls required container images
- Creates and starts containers
- Monitors health using liveness and readiness probes
- Sends status updates to the API server

> **Health Checks**: If a container fails a liveness probe, it is restarted. If it fails a readiness probe, it's temporarily removed from service.

---
### How Does the Kubelet Pull Images?

The Kubelet interacts with the container runtime using the **Container Runtime Interface (CRI)**, a gRPC-based API. It connects to the runtime via a Unix socket:

```bash
/run/containerd/containerd.sock       # for containerd
/var/run/crio/crio.sock              # for CRI-O
```

You can configure this using the `--container-runtime-endpoint` flag:

```bash
kubelet --container-runtime-endpoint=unix:///run/containerd/containerd.sock
```

To verify the connection:

```bash
sudo crictl info
```

To check which socket the Kubelet is using:

```bash
ps aux | grep kubelet
```

**Flow Summary**:

1. Kubelet asks CRI to create a Pod sandbox (network namespace).
2. CRI uses `runc` (or similar) to create it.
3. Kubelet requests image download via CRI.
4. Kubelet instructs CRI to start the container.
5. Kubelet monitors the container via CRI.

---
### Pod Lifecycle and Maintenance

Once running, a Pod goes through different phases:

|Phase|Description|
|---|---|
|`Pending`|Accepted by the cluster, waiting to be scheduled or for images to download.|
|`Running`|Scheduled to a node, with at least one running container.|
|`Succeeded`|All containers exited successfully and won’t restart.|
|`Failed`|All containers exited, and at least one failed.|
|`Unknown`|Pod state cannot be determined, often due to communication issues.|

Kubernetes handles restarts via `restartPolicy`:

- **Initial crash**: Container restarts immediately.
- **Repeated crashes**: Backoff delays increase exponentially.
- **CrashLoopBackOff**: Indicates repeated failures with backoff in place.
- **Backoff reset**: If a container runs successfully for a set time, backoff resets.

>**Evicted Pods**: Sometimes, Pods are terminated and marked as `Evicted` due to node-level resource pressure (e.g., memory or disk). This is not a failure in the Pod itself, but a protective measure by the kubelet. These Pods are removed to reclaim resources, and depending on the controller (e.g., Deployment or StatefulSet), they are typically recreated on another node.

---
### What about kube-proxy?

We talked about other components of Kubernetes but what about kube-proxy? `kube-proxy` does not **directly** involved during the `kubectl apply`But it **plays a critical role afterward**, once your Deployment results in Pods and **Services** being created.

kube-proxy is the component that manages networking rules on each node to route traffic to the appropriate backend Pods. It handles:

- **Virtual IPs (ClusterIPs)** for Services
- **NAT rules** using `iptables`, `ipvs`, or `ebpf` (depending on config)
- **Load balancing** between Pods behind a Service
- **NodePort exposure** for Services if configured

After the Pods from your Deployment are up and running and a Service is created, kube-proxy:

- Detects the creation of the Service (via watching the API Server).
- Sets up routing rules (e.g., iptables) so that:
- Requests to the Service IP and port are load balanced to backend Pods.
- Continuously watches for changes in Pod IPs and updates routing accordingly.

> **Note**: For Pod-level networking (IP assignment, Pod-to-Pod reachability), Kubernetes uses **CNI plugins** (e.g., Calico, Flannel, Cilium). kube-proxy sits on top of that for Service abstraction.

---
### Events and Observability

Kubernetes is event-driven. During every step — from applying manifests to scheduling, pulling images, or crashing — Kubernetes components emit **Events**. These are stored in the API server and visible via:

```bash
kubectl describe pod <pod-name>
```

You'll often see logs like:

- "Successfully assigned pod..."
- "Started container..."
- "Back-off restarting failed container..."

These are critical for debugging the lifecycle of resources.

---
### To summarize

This entire process demonstrates how Kubernetes self-heals, maintains state, and abstracts away the operational complexity of deploying and running containers at scale.

Think of it like this:

- You (`kubectl`) send a construction plan to the city (`API server`).
- Builders (`controllers`, `scheduler`, `kubelet`) build the house (`Pods`, `ReplicaSets`).
- kube-proxy is the traffic engineer: it comes in after the houses are built and makes sure people (network traffic) know how to get there.

And all of it is made possible by the **feedback loop** between your declaration, the control plane, and the controllers that keep things aligned — always striving for **desired state = actual state**.