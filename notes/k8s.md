# Kubernetes

![Cheat Sheet](https://raw.githubusercontent.com/aint/interview/refs/heads/main/k8s_cheatsheet.png)

**Kubernetes (K8s)** is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.

A Kubernetes cluster consists of a **control plane** and **worker nodes**. The control plane's components make global decisions about the cluster (for example, scheduling), as well as detecting and responding to cluster events (for example, starting up a new pod when a Deployment's replicas field is unsatisfied). The worker nodes run containerized applications.

## Table of Contents
1. [Architecture Components](#architecture-components)
2. [Workloads](#workloads)
3. [Workload Management](#workload-management)
4. [Scaling](#scaling)
5. [Networking](#networking)
6. [Storage & Config](#storage--config)
7. [Quick Reference](#quick-reference)
8. [Cheat Sheet](#cheat-sheet)

---

# Architecture Components

## Namespace

Logical isolation within a cluster. Resources are scoped to namespaces (except some cluster-level resources).

**Default namespaces**:
- `default`: Default namespace for resources
- `kube-system`: Kubernetes system components
- `kube-public`: Publicly accessible data
- `kube-node-lease`: Node heartbeat objects

## Node

A worker machine (VM or physical machine) that runs pods (your workload). Nodes have:
- **kubelet**: Agent running on each node
- **kube-proxy**: Network proxy for services
- **Container runtime**: Software responsible for running containers (containerd, Docker, etc.)

## Leases

Leases are used for:
- **Node heartbeats**: Nodes use Lease objects in `kube-node-lease` namespace to report their health. More efficient than updating Node objects directly.
- **Leader election**: Components use Leases to coordinate leader election (e.g., kube-controller-manager, kube-scheduler). Only the leader performs operations.


## Labels and Selectors

- **Labels**: Key-value pairs attached to objects for organization and selection
- **Selectors**: Used to identify a set of objects matching specific labels

Example: `app=frontend, environment=production`

---

# Workloads

A workload is an application running on Kubernetes. Whether your workload is a single component or several that work together, on Kubernetes you run it inside a set of pods. In Kubernetes, a Pod represents a set of running containers on your cluster.

## Pod

The smallest deployable unit in Kubernetes. A pod contains one or more **containers** that share:
- Storage volumes
- Network namespace (same IP address)
- Resources (CPU, memory limits)

**Key characteristics**:
- Ephemeral: Pods can be created, destroyed, and recreated
- Each pod gets a unique IP address
- Containers in a pod are co-located and co-scheduled

### Pod Lifecycle

**Pod states**:
- **Pending**: Pod accepted, but containers not created.
- **Running**: The Pod has been bound to a node, and all of the containers have been created. At least one container is still running, or is in the process of starting or restarting.
- **Succeeded**: All containers in the Pod have terminated in success, and will not be restarted.
- **Failed**: All containers in the Pod have terminated, and at least one container has terminated in failure.
- **Unknown**: Pod state cannot be obtained.

**Container states**:
- **Waiting**: Container is waiting to start (e.g., pulling image, waiting for init container)
- **Running**: Container is executing
- **Terminated**: Container has stopped (success or failure)

### Probes

Health checks for containers:
- **Liveness probe**: Determines if container is running. If fails, container is restarted.
- **Readiness probe**: Determines if container is ready to serve traffic. If fails, pod removed from service endpoints.
- **Startup probe**: Allows slow-starting containers to avoid being killed by liveness probes.

**Probe types**: HTTP, TCP socket, Exec command

### Init Containers

Containers that run before app containers in a pod. Useful for:
- Setup tasks
- Dependency checks
- Data synchronization

### Multi-Container Pods

Use multiple containers in a pod for:
- **Sidecar pattern**: Helper container (logging, monitoring, proxy)
- **Adapter pattern**: Normalizes output from main container
- **Ambassador pattern**: Proxies localhost connections to external services

### Resource Requests and Limits

**Requests**: Guaranteed resources for a container (used for scheduling)
- CPU: `100m` (0.1 core) or `1` (1 core)
- Memory: `128Mi`, `1Gi`

**Limits**: Maximum resources a container can use
- CPU: Container throttled if exceeds
- Memory: Container killed (OOMKilled) if exceeds

### Pod Quality of Service Classes

QoS classes determine pod priority for eviction when nodes run out of resources:
- **Guaranteed**: All containers have requests = limits. Highest priority, last to be evicted.
- **Burstable**: At least one container has request < limit. Medium priority.
- **BestEffort**: No requests or limits set. Lowest priority, first to be evicted.

---

# Workload Management

### Deployment

Manages a set of identical pods (replica set). Provides:
- **Declarative updates**: Describes desired state
- **Rolling updates**: Updates pods gradually
- **Rollback**: Reverts to previous version
- **Scaling**: Changes number of replicas

**Key fields**:
- `replicas`: Number of pod replicas
- `selector`: Labels to identify managed pods
- `template`: Pod template for creating pods
- `strategy`: Update strategy (RollingUpdate or Recreate)

**Deployment Strategies**:
- **Rolling Update** (default): Gradually update pods, maintains availability
- **Recreate**: Terminate all old pods before creating new ones (downtime)
- **Blue/Green**: Run two complete environments, switch traffic (requires external tooling)
- **Canary**: Gradually route traffic to new version (requires external tooling)

### ReplicaSet

Ensures a specified number of pod replicas are running. Generally managed by Deployments (you create Deployments, which create ReplicaSets).

### StatefulSet

Manages stateful applications with stable network identities and persistent storage.
- Provides stable, unique network identifiers
- Provides stable, persistent storage per pod
- Ordered, graceful deployment and scaling

**Use cases**: Databases, stateful applications requiring unique identities.

### DaemonSet

Ensures a copy of a pod runs on all (or selected) nodes.
- Automatically creates/removes pods when nodes are added/removed
- Common use cases: Log collectors, monitoring agents, network plugins

### Job

Creates one or more pods and ensures a specified number complete successfully.
- Runs until completion (not continuous)
- Useful for batch processing, one-time tasks

### CronJob

Creates Jobs on a time-based schedule (like Unix cron).
- Recurring tasks at specified times

---

# Scaling

## Horizontal Pod Autoscaler (HPA)

Automatically scales the number of pods based on observed metrics.

**Triggers**:
- CPU utilization (default)
- Memory utilization
- Custom metrics

**Example**: Scale between 2-10 replicas when CPU > 70%

## Vertical Pod Autoscaler (VPA)

Automatically adjusts CPU and memory requests/limits for containers based on historical usage.

**Limitations**:
- Cannot be used with HPA on CPU/memory (conflict)
- Requires VPA controller to be installed (not part of core Kubernetes)
- Recommends or applies new resource values

## Cluster Autoscaler

Automatically adjusts the number of nodes in the cluster based on demand.

**Key points**:
- Cloud provider specific (AWS, GCP, Azure)
- Adds nodes when pods can't be scheduled (pending pods)
- Removes nodes when underutilized
- Works with HPA to scale both pods and nodes

---

# Networking

Kubernetes networking provides:
- **Ingress**: External HTTP/HTTPS access to services
- **Service discovery**: DNS-based service discovery within the cluster
- **Load balancing**: Distribute traffic across pod replicas
- **Network policies**: Control pod-to-pod communication


## Service

Abstraction that defines a logical set of pods and a policy to access them. Provides stable IP address and DNS name. Services automatically load balance traffic across pod endpoints.

**Service Types**:
- **ClusterIP** (default): Exposes service on cluster-internal IP. Only accessible within the cluster.
- **NodePort**: Exposes service on each node's IP at a static port (30000-32767). Accessible from outside the cluster.
- **LoadBalancer**: Exposes service externally via cloud provider's load balancer. Extends NodePort.
- **ExternalName**: Maps service to external DNS name. Returns CNAME record.

**Load Balancing**: Services use kube-proxy to distribute traffic across healthy pod endpoints. Default algorithm is round-robin (implementation varies by proxy mode).

## Ingress

Manages external HTTP/HTTPS access to services. Provides:
- URL-based routing
- SSL/TLS termination
- Load balancing

Requires an **Ingress Controller** (e.g., NGINX, Traefik, Kong).

## Network Policies

Define rules for pod-to-pod communication (firewall rules).
- **PodSelector**: Selects pods the policy applies to
- **PolicyTypes**: Ingress, Egress, or both
- **Rules**: Define allowed traffic (from/to pods, namespaces, IP blocks)


## Service Discovery

### DNS-based Service Discovery

Kubernetes provides DNS for services:
- **Service DNS**: `<service-name>.<namespace>.svc.cluster.local`
- Short form: `<service-name>` (same namespace) or `<service-name>.<namespace>` (different namespace)

### Service Types for Discovery

- **ClusterIP**: Internal service discovery
- **NodePort**: Accessible via `<NodeIP>:<NodePort>`
- **LoadBalancer**: External IP provided by cloud provider
- **Headless Service** (`clusterIP: None`): Returns individual pod IPs (useful for StatefulSets)

---

# Storage & Config

## Volume

Volumes are mounted into pods at specified paths. Containers in a pod can share volumes.

- **Ephemeral volumes** (tied to pod lifecycle):
  - **emptyDir**: Temporary storage, cleared when pod removed
  - **ConfigMap/Secret**: Mount configuration data into pods
- **Persistent volumes** (survive pod restarts)

## ConfigMap

Stores non-confidential configuration data in key-value pairs.
- Can be mounted as files or exposed as environment variables
- Changes don't require pod restart (if mounted as volume)

##  Secret

Stores sensitive data (passwords, tokens, keys).
- Similar to ConfigMap but for sensitive data
- Base64 encoded (not encrypted - don't commit to git)

---

## Quick Reference

**Common Interview Questions**:
- What is the difference between a Deployment and a ReplicaSet? (Deployment manages ReplicaSets, provides updates/rollbacks)
- How does Kubernetes handle pod failures? (Controller restarts pods, reschedules if node fails)
- What is the difference between a Service and an Ingress? (Service provides internal LB, Ingress provides HTTP routing)
- How do you achieve zero-downtime deployments? (Rolling updates with proper readiness probes)
- What is the difference between requests and limits? (Requests for scheduling, limits for capping)
- How does service discovery work? (DNS-based, service name resolves to cluster IP)
- What is a StatefulSet used for? (Stateful apps requiring stable network identity and storage)
- How does HPA work? (Scales pods based on metrics like CPU/memory utilization)
- What is the difference between ConfigMap and Secret? (ConfigMap for config data, Secret for sensitive data)
- How does Kubernetes schedule pods? (Scheduler assigns pods to nodes based on resource availability and constraints)

**Key Differences**:
- **Pod vs Container**: Pod can have multiple containers sharing network/storage
- **Deployment vs StatefulSet**: Deployment for stateless apps, StatefulSet for stateful apps
- **Service vs Ingress**: Service for internal LB, Ingress for external HTTP routing
- **ConfigMap vs Secret**: ConfigMap for non-sensitive config, Secret for sensitive data

