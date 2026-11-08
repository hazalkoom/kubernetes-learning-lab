## Kubernetes System Components

### Control Plane
- kube-apiserver: Kubernetes API entry point.
- etcd: Persistent key-value store for cluster state.
- kube-scheduler: Selects nodes for unscheduled Pods.
- kube-controller-manager: Runs controllers that reconcile desired state.

### Node Components
- kubelet: Node agent responsible for managing Pods.
- kube-proxy: Implements Service networking rules in common configurations.
- Container runtime: Runs containers.

### Cluster Services
- CoreDNS: Provides DNS-based service discovery.
- CNI: Provides Pod networking.

## Pod Management Types

### Static Pods
- Managed directly by kubelet.
- Defined using local manifest files.
- Commonly used for Kubernetes control-plane components.
- Not managed by Deployments.

### Deployments
- Manage application replicas.
- Use ReplicaSets to maintain the requested number of Pods.
- Support rolling updates and rollbacks.

### DaemonSets
- Ensure a Pod runs on every eligible node.
- Commonly used for networking and node-level agents.

## Workload Management Mechanisms

### Deployment

A Deployment manages application workloads through ReplicaSets.

Hierarchy:

Deployment → ReplicaSet → Pods

It supports scaling, rolling updates, and rollbacks.

Example: CoreDNS runs as a Deployment with two replicas.

### DaemonSet

A DaemonSet ensures that a Pod runs on every eligible node.

Examples:
- kube-proxy
- kindnet

With one eligible node, each currently has one Pod.

### Static Pods

Static Pods are managed directly by kubelet using local manifest files.

In our Kind control-plane node, the manifests are located at:

`/etc/kubernetes/manifests/`

Our cluster uses Static Pods for:
- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler

## Resource Management

- Requests: Resources requested by a container and used by the scheduler when placing Pods.
- Limits: Maximum resource usage enforced for supported resources.

## Health Probes

- Liveness probe: Determines whether a container should be restarted.
- Readiness probe: Determines whether a container is ready to receive traffic.

## First Application Deployment

### Pod
The smallest deployable unit in Kubernetes.
Containers inside a Pod share networking and configured storage.

### Deployment
Manages application replicas through ReplicaSets.
Supports scaling, rolling updates, and rollbacks.

### ReplicaSet
Maintains the requested number of matching Pods.

### Service
Provides a stable network endpoint for accessing matching Pods.

### Namespace
Logically separates Kubernetes resources within a cluster.

### Declarative Management
We define desired state in YAML and apply it using kubectl.
Kubernetes controllers continuously reconcile actual state with desired state.

### Resource Requests and Limits
Requests influence scheduling.
Limits restrict container resource consumption.

### Health Probes
- Readiness: Determines whether a Pod should receive traffic.
- Liveness: Determines whether a container should be restarted.

### Self-Healing
When a managed Pod is deleted, its ReplicaSet detects the missing replica and creates a replacement.

### Port Forwarding
kubectl port-forward creates a temporary connection from a local port to a Kubernetes resource.