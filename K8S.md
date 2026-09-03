# Kubernetes Master Notes: Architecture, Workloads & Production Deployment

## Complete Kubernetes Guide for DevOps Engineers

---

# 1. What is Kubernetes?

**Kubernetes (K8s)** is an open-source container orchestration platform used to deploy, manage, scale, and maintain containerized applications.

Docker can run containers:

```text
Docker
  ↓
Runs containers
```

Kubernetes manages many containers across one or more machines:

```text
                    Kubernetes
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
       Node 1         Node 2         Node 3
          │              │              │
      Containers      Containers      Containers
```

Kubernetes can automatically:

* Deploy applications
* Restart failed containers
* Scale applications
* Perform rolling updates
* Roll back failed deployments
* Distribute workloads across nodes
* Provide service discovery
* Load-balance traffic
* Manage persistent storage
* Manage application configuration
* Enforce access control

---

# 2. Why Do We Need Kubernetes?

Suppose you have one application:

```text
Application
    ↓
1 Docker container
    ↓
1 Server
```

Docker alone may be sufficient.

Now imagine:

```text
100 containers
10 servers
Multiple applications
Thousands of users
```

Manually managing them becomes difficult.

Kubernetes automates this management.

### Without Kubernetes

```text
Developer
   ↓
Build Docker image
   ↓
SSH into server
   ↓
Start container
   ↓
Monitor container
   ↓
Restart if crashed
   ↓
Scale manually
```

### With Kubernetes

```text
Developer
    ↓
Container Image
    ↓
Kubernetes
    ↓
Deploy
    ↓
Schedule
    ↓
Monitor
    ↓
Restart
    ↓
Scale
    ↓
Update
```

---

# 3. Kubernetes vs Docker

| Docker                              | Kubernetes                       |
| ----------------------------------- | -------------------------------- |
| Container platform                  | Container orchestration platform |
| Runs containers                     | Manages containers               |
| Good for individual/small workloads | Good for distributed workloads   |
| Docker CLI                          | kubectl                          |
| Docker Compose                      | Kubernetes manifests/Helm        |
| One/few machines                    | Multiple nodes                   |
| Manual scaling                      | Automated scaling possible       |
| Basic networking                    | Advanced networking              |
| Basic container lifecycle           | Application orchestration        |

They are not necessarily competitors.

A common architecture is:

```text
Application
     ↓
Dockerfile
     ↓
Docker Image
     ↓
Container Registry
     ↓
Kubernetes
     ↓
Pods
```

Modern Kubernetes clusters use container runtimes such as **containerd** or **CRI-O** rather than requiring Docker Engine.

---

# 4. Kubernetes Architecture

A Kubernetes cluster consists primarily of:

```text
                 Kubernetes Cluster
                        │
          ┌─────────────┴─────────────┐
          │                           │
     Control Plane                Worker Nodes
          │                           │
    ┌─────┼─────┐              ┌──────┼──────┐
    ↓     ↓     ↓              ↓      ↓      ↓
 API   Scheduler Controller   Pod    Pod    Pod
Server          Manager
    │
   etcd
```

---

# 5. Control Plane

The **Control Plane** manages the Kubernetes cluster.

Major components:

1. API Server
2. etcd
3. Scheduler
4. Controller Manager
5. Cloud Controller Manager

---

# 6. API Server

The **kube-apiserver** is the central communication point for Kubernetes.

When you run:

```bash
kubectl get pods
```

the request goes approximately:

```text
kubectl
   ↓
API Server
   ↓
Kubernetes state
```

The API Server:

* Authenticates requests
* Authorizes requests
* Validates objects
* Stores/retrieves cluster state
* Communicates with other Kubernetes components

---

# 7. etcd

**etcd** is Kubernetes' distributed key-value store.

It stores important cluster state such as:

* Pods
* Deployments
* Services
* Configurations
* Secrets
* Nodes
* Cluster metadata

Think:

```text
Kubernetes
     ↓
   etcd
     ↓
Cluster state
```

If the control-plane state stored in etcd is lost, cluster recovery can become difficult.

Therefore, production Kubernetes environments require proper etcd backup strategies.

---

# 8. Scheduler

The **kube-scheduler** decides which worker node should run a newly created Pod.

Example:

```text
New Pod
   ↓
Scheduler
   ↓
Node 1? ❌
Node 2? ✅
Node 3? ❌
   ↓
Pod scheduled on Node 2
```

It considers factors such as:

* CPU availability
* Memory availability
* Resource requests
* Node selectors
* Affinity/anti-affinity
* Taints and tolerations
* Other scheduling constraints

---

# 9. Controller Manager

The Controller Manager runs various controllers.

Controllers continuously compare:

```text
Desired State
      vs
Current State
```

Example:

You specify:

```yaml
replicas: 3
```

But only two Pods are running.

The controller notices:

```text
Desired = 3
Current = 2
```

and creates another Pod.

This is one of the most important Kubernetes concepts:

> Kubernetes continuously attempts to make the actual state match the desired state.

---

# 10. Worker Node

Worker nodes run application workloads.

A node generally contains:

* kubelet
* container runtime
* kube-proxy or equivalent networking components
* Pods

Example:

```text
Worker Node
│
├── kubelet
├── container runtime
├── networking components
│
├── Pod
│   └── Container
│
├── Pod
│   └── Container
│
└── Pod
    └── Container
```

---

# 11. kubelet

The **kubelet** runs on each worker node.

Its job is to:

* Communicate with the API Server
* Ensure assigned Pods are running
* Monitor containers
* Report node/Pod status

Think:

```text
API Server
     ↓
  kubelet
     ↓
Containers
```

---

# 12. Container Runtime

The container runtime is responsible for actually running containers.

Common runtimes include:

* containerd
* CRI-O

Kubernetes communicates with runtimes through the **Container Runtime Interface (CRI)**.

---

# 13. kube-proxy

kube-proxy traditionally helps implement Kubernetes Service networking on nodes.

It manages networking rules that allow traffic to reach Services and their backend Pods.

Modern Kubernetes networking implementations can vary, particularly with eBPF-based networking solutions.

For learning purposes:

```text
Client
  ↓
Service
  ↓
Networking rules
  ↓
Pod
```

---

# 14. Kubernetes Objects

Kubernetes manages resources called **objects**.

Common objects include:

```text
Pod
Deployment
ReplicaSet
Service
ConfigMap
Secret
Namespace
Ingress
StatefulSet
DaemonSet
Job
CronJob
PersistentVolume
PersistentVolumeClaim
ServiceAccount
Role
RoleBinding
```

Most are defined using YAML.

---

# 15. Declarative Approach

Kubernetes primarily follows a declarative model.

Instead of saying:

```text
Start three containers.
```

you declare:

```yaml
replicas: 3
```

Kubernetes determines how to achieve that state.

```text
Desired State
      ↓
Kubernetes Controllers
      ↓
Actual State
```

---

# 16. Kubernetes YAML

A typical manifest contains:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 3
```

Four important sections are:

```text
apiVersion
kind
metadata
spec
```

---

# 17. apiVersion

Defines the API version used by the object.

Examples:

```yaml
apiVersion: v1
```

```yaml
apiVersion: apps/v1
```

```yaml
apiVersion: networking.k8s.io/v1
```

---

# 18. kind

Defines the object type.

Examples:

```yaml
kind: Pod
```

```yaml
kind: Deployment
```

```yaml
kind: Service
```

---

# 19. metadata

Contains identifying information.

```yaml
metadata:
  name: my-app
  labels:
    app: my-app
```

---

# 20. spec

Defines the desired configuration.

```yaml
spec:
  replicas: 3
```

---

# 21. Pods

A **Pod** is the smallest deployable unit in Kubernetes.

A Pod can contain one or more containers.

Most applications use:

```text
1 Pod
  ↓
1 main container
```

But a Pod can contain multiple tightly coupled containers:

```text
Pod
├── Application container
└── Sidecar container
```

Containers inside the same Pod share:

* Network namespace
* Pod IP
* Local volumes

---

# 22. Basic Pod YAML

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-pod

spec:
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
```

Create:

```bash
kubectl apply -f pod.yaml
```

Check:

```bash
kubectl get pods
```

Detailed information:

```bash
kubectl describe pod nginx-pod
```

---

# 23. Pod Lifecycle

A Pod can move through phases such as:

```text
Pending
   ↓
Running
   ↓
Succeeded
```

or:

```text
Pending
   ↓
Running
   ↓
Failed
```

The container inside the Pod can also restart independently according to its restart policy.

---

# 24. Pod Restart Policy

Possible values include:

```yaml
restartPolicy: Always
```

```yaml
restartPolicy: OnFailure
```

```yaml
restartPolicy: Never
```

For Pods managed by a Deployment, the normal pattern is to use the default `Always` behavior.

---

# 25. Multi-Container Pods

Example:

```yaml
spec:
  containers:

    - name: app
      image: myapp:1.0

    - name: sidecar
      image: busybox
```

Use multiple containers in a Pod when the containers are tightly coupled and need to share networking/storage or lifecycle.

Do not put unrelated applications into the same Pod simply because they are part of the same project.

---

# 26. Init Containers

Init containers run before application containers.

```text
Pod starts
   ↓
Init Container
   ↓
Initialization
   ↓
Main Container
```

Example:

```yaml
initContainers:
  - name: init
    image: busybox
    command:
      - sh
      - -c
      - echo "Initializing..."
```

Common uses:

* Initialization
* Waiting for prerequisites
* Preparing files
* Running setup logic

---

# 27. Deployment

A **Deployment** manages stateless application Pods.

Instead of manually creating Pods:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

---

# 28. Deployment Architecture

```text
Deployment
     ↓
ReplicaSet
     ↓
 ┌───┼───┐
 ↓   ↓   ↓
Pod Pod Pod
```

The Deployment controls the ReplicaSet.

The ReplicaSet ensures the desired number of Pods exists.

---

# 29. Scaling Deployment

Current replicas:

```bash
kubectl get deployment
```

Scale to five:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Or change:

```yaml
replicas: 5
```

and apply:

```bash
kubectl apply -f deployment.yaml
```

---

# 30. Rolling Updates

Suppose you currently run:

```text
nginx:1.25
```

and update to:

```text
nginx:1.26
```

Kubernetes can gradually replace old Pods with new Pods.

```text
Old Pods
  ↓
New Pods created
  ↓
Old Pods removed
  ↓
New version running
```

This is a rolling update.

---

# 31. Rollback

Check rollout:

```bash
kubectl rollout status deployment nginx-deployment
```

View history:

```bash
kubectl rollout history deployment nginx-deployment
```

Rollback:

```bash
kubectl rollout undo deployment nginx-deployment
```

---

# 32. ReplicaSet

A ReplicaSet maintains a specified number of Pod replicas.

Example:

```text
Desired: 3

Running:
Pod 1
Pod 2
Pod 3
```

If one disappears:

```text
Pod 1
Pod 2
Pod 3 ❌
```

ReplicaSet creates another:

```text
Pod 1
Pod 2
Pod 4
```

Usually, you don't manage ReplicaSets directly when using Deployments.

---

# 33. Service

Pods are ephemeral.

Their IP addresses can change.

A **Service** provides a stable network endpoint for a set of Pods.

```text
             Service
                │
       ┌────────┼────────┐
       ↓        ↓        ↓
      Pod      Pod      Pod
```

---

# 34. ClusterIP

Default Service type:

```yaml
type: ClusterIP
```

Used for internal cluster communication.

Example:

```text
Frontend
   ↓
backend-service
   ↓
Backend Pods
```

The frontend doesn't need to know individual Pod IP addresses.

---

# 35. NodePort

NodePort exposes a Service through a port on each node.

```yaml
type: NodePort
```

Concept:

```text
User
 ↓
Node IP:NodePort
 ↓
Service
 ↓
Pod
```

NodePort is useful for learning and some simple setups, but production applications often use LoadBalancer or Ingress instead.

---

# 36. LoadBalancer

```yaml
type: LoadBalancer
```

In a cloud environment, this typically provisions or integrates with an external load balancer.

```text
Internet
   ↓
Cloud Load Balancer
   ↓
Kubernetes Service
   ↓
Pods
```

---

# 37. Service Example

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-service

spec:
  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80

  type: ClusterIP
```

Important:

```text
port
   ↓
Service port

targetPort
   ↓
Container/Pod port
```

---

# 38. Labels and Selectors

Labels identify Kubernetes resources.

```yaml
labels:
  app: backend
```

A Service can select them:

```yaml
selector:
  app: backend
```

Architecture:

```text
Service
 selector:
 app=backend
       ↓
┌──────┼──────┐
↓      ↓      ↓
Pod    Pod    Pod
app=   app=   app=
backend backend backend
```

Labels are fundamental to Kubernetes.

---

# 39. Namespaces

Namespaces logically separate resources.

Example:

```text
Cluster
│
├── development
│   ├── Pods
│   └── Services
│
├── staging
│   ├── Pods
│   └── Services
│
└── production
    ├── Pods
    └── Services
```

Create:

```bash
kubectl create namespace dev
```

Use:

```bash
kubectl get pods -n dev
```

---

# 40. ConfigMap

ConfigMaps store non-sensitive configuration.

Example:

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: app-config

data:
  APP_ENV: production
  LOG_LEVEL: info
```

Pod:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

---

# 41. Secret

Secrets are designed for sensitive configuration such as:

* Passwords
* Tokens
* API keys
* Credentials

Example:

```yaml
apiVersion: v1
kind: Secret

metadata:
  name: app-secret

type: Opaque

stringData:
  DB_USER: admin
  DB_PASSWORD: example-password
```

Important:

> Kubernetes Secret objects are not automatically equivalent to strong external secret-management systems. Protect access to them and consider encryption at rest and external secret managers for production.

---

# 42. ConfigMap vs Secret

| ConfigMap                   | Secret                                                 |
| --------------------------- | ------------------------------------------------------ |
| Non-sensitive configuration | Sensitive configuration                                |
| URLs                        | Passwords                                              |
| Feature flags               | Tokens                                                 |
| Application settings        | Credentials                                            |
| Plain configuration         | Encoded by default in manifests/storage representation |

Never commit real production secrets to Git.

---

# 43. Environment Variables

Example:

```yaml
env:
  - name: APP_ENV
    value: production
```

From ConfigMap:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

From Secret:

```yaml
envFrom:
  - secretRef:
      name: app-secret
```

---

# 44. Volumes

Containers are generally ephemeral.

If a container is removed, data inside its writable container filesystem may disappear.

Volumes provide storage that can outlive an individual container.

Example:

```yaml
volumes:
  - name: app-data
    emptyDir: {}
```

Mount:

```yaml
volumeMounts:
  - name: app-data
    mountPath: /data
```

---

# 45. PersistentVolume

A **PersistentVolume (PV)** represents storage available to the cluster.

```text
Storage
   ↓
PersistentVolume
```

---

# 46. PersistentVolumeClaim

A **PersistentVolumeClaim (PVC)** requests storage.

```text
Application
    ↓
PVC
    ↓
PV
    ↓
Actual Storage
```

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: app-pvc

spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi
```

---

# 47. StorageClass

StorageClasses allow dynamic storage provisioning.

```text
PVC
 ↓
StorageClass
 ↓
Provisioner
 ↓
Storage
```

Cloud examples can dynamically create block or file storage according to the configured CSI driver.

---

# 48. StatefulSet

Use **StatefulSet** for stateful applications that require stable identity or storage.

Common examples:

* Databases
* Kafka
* ZooKeeper
* Stateful distributed systems

Deployment:

```text
Pod
Pod
Pod
```

StatefulSet:

```text
app-0
app-1
app-2
```

Pods have stable identities.

---

# 49. StatefulSet Example

```yaml
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: web

spec:
  serviceName: web
  replicas: 3

  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web

    spec:
      containers:
        - name: nginx
          image: nginx:alpine
```

---

# 50. Deployment vs StatefulSet

| Deployment                | StatefulSet                             |
| ------------------------- | --------------------------------------- |
| Stateless applications    | Stateful applications                   |
| Pods are interchangeable  | Pods have stable identity               |
| Typical web/API workloads | Databases/distributed systems           |
| Random Pod identity       | Ordered/stable identity                 |
| Common default for APIs   | Used when stateful behavior requires it |

---

# 51. DaemonSet

A DaemonSet ensures that a Pod runs on selected nodes, commonly one per eligible node.

Typical uses:

* Log collectors
* Node monitoring agents
* Networking agents
* Security agents

Architecture:

```text
Node 1 → Agent
Node 2 → Agent
Node 3 → Agent
Node 4 → Agent
```

---

# 52. Job

A Job is designed for a task that should complete.

Example:

```text
Start
 ↓
Process data
 ↓
Complete
```

Example:

```yaml
apiVersion: batch/v1
kind: Job

metadata:
  name: database-migration

spec:
  template:
    spec:
      restartPolicy: Never

      containers:
        - name: migration
          image: myapp:1.0
          command:
            - ./migrate
```

---

# 53. CronJob

CronJob runs Jobs on a schedule.

Example:

```yaml
apiVersion: batch/v1
kind: CronJob

metadata:
  name: backup

spec:
  schedule: "0 2 * * *"

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
            - name: backup
              image: backup-image:1.0
```

The schedule means:

```text
Every day at 02:00
```

---

# 54. Ingress

Ingress provides HTTP/HTTPS routing into a Kubernetes cluster.

Concept:

```text
Internet
    ↓
Ingress
    ↓
┌─────────────┬─────────────┐
↓             ↓
Frontend      Backend
Service       Service
```

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: app-ingress

spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

Ingress requires an **Ingress Controller**.

Modern Kubernetes environments may also use the newer **Gateway API** for more advanced traffic-management use cases.

---

# 55. NetworkPolicy

NetworkPolicy controls allowed network traffic between Pods and/or external endpoints, depending on the networking implementation.

Example concept:

```text
Frontend → Backend ✅
Frontend → Database ❌
Backend → Database ✅
```

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: backend-policy

spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
```

NetworkPolicy enforcement depends on the cluster's network plugin supporting it.

---

# 56. Resource Requests

Requests tell Kubernetes how much resource a container needs for scheduling.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
```

Meaning approximately:

```text
CPU    → 0.25 CPU
Memory → 256 MiB
```

The scheduler uses requests when deciding where to place Pods.

---

# 57. Resource Limits

Limits define an upper bound on resource consumption.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"

  limits:
    cpu: "500m"
    memory: "512Mi"
```

Concept:

```text
Requests → scheduling expectation
Limits   → maximum allowed resource usage
```

Exact CPU/memory behavior depends on the resource and runtime.

---

# 58. Liveness Probe

A liveness probe checks whether an application should be restarted.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080

  initialDelaySeconds: 10
  periodSeconds: 10
```

If the container repeatedly fails its liveness check, kubelet can restart it.

---

# 59. Readiness Probe

Readiness determines whether a Pod should receive traffic.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080

  periodSeconds: 5
```

Important difference:

```text
Liveness
   ↓
Should container be restarted?

Readiness
   ↓
Should container receive traffic?
```

---

# 60. Startup Probe

Startup probes are useful for applications that take a long time to initialize.

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080

  failureThreshold: 30
  periodSeconds: 10
```

This prevents liveness checks from restarting an application while it is still starting.

---

# 61. Scheduling

Kubernetes decides where Pods should run.

Important scheduling concepts:

* Requests
* Node selectors
* Affinity
* Anti-affinity
* Taints
* Tolerations
* Topology spread constraints

---

# 62. NodeSelector

Simple node selection.

Label node:

```bash
kubectl label nodes node1 disktype=ssd
```

Pod:

```yaml
nodeSelector:
  disktype: ssd
```

The Pod can only be scheduled on matching nodes.

---

# 63. Node Affinity

Affinity provides more flexible node-selection rules.

Example:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: disktype
              operator: In
              values:
                - ssd
```

---

# 64. Taints and Tolerations

A taint can prevent ordinary Pods from being scheduled on a node.

```text
Node
 ↓
Taint
 ↓
Only Pods with matching toleration can schedule
```

Example:

```bash
kubectl taint nodes node1 dedicated=database:NoSchedule
```

Pod:

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: database
    effect: NoSchedule
```

Common use:

```text
Database nodes
GPU nodes
Special-purpose nodes
```

---

# 65. Horizontal Pod Autoscaler

HPA automatically adjusts the number of Pod replicas based on metrics.

```text
Low traffic
   ↓
2 Pods

High traffic
   ↓
5 Pods

Very high traffic
   ↓
10 Pods
```

Concept:

```yaml
minReplicas: 2
maxReplicas: 10
```

HPA commonly uses CPU/memory utilization or custom/external metrics depending on cluster configuration.

---

# 66. Vertical Pod Autoscaler

VPA can recommend or adjust resource requests/limits for Pods.

Concept:

```text
Application uses more memory
       ↓
VPA analyzes usage
       ↓
Recommends larger memory request
```

VPA is not simply a replacement for HPA; they solve different scaling problems.

---

# 67. Cluster Autoscaler

Cluster Autoscaler operates at the **node level**.

Example:

```text
Pods cannot be scheduled
        ↓
Cluster needs more capacity
        ↓
Add node
```

When nodes become unnecessary, it can remove them subject to its configured rules.

---

# 68. HPA vs VPA vs Cluster Autoscaler

| Tool               | Scales                       |
| ------------------ | ---------------------------- |
| HPA                | Number of Pods               |
| VPA                | Pod resource requests/limits |
| Cluster Autoscaler | Number of nodes              |

Think:

```text
HPA
Pod count ↑

VPA
Pod resources ↑

Cluster Autoscaler
Node count ↑
```

---

# 69. Service Discovery

Kubernetes provides DNS-based service discovery.

If you have:

```text
Service:
backend
```

another Pod can generally reach it using:

```text
backend
```

within the same namespace.

A fully qualified service DNS name can look like:

```text
backend.default.svc.cluster.local
```

---

# 70. Kubernetes DNS

Typical structure:

```text
service.namespace.svc.cluster.local
```

Example:

```text
backend.production.svc.cluster.local
```

This allows applications to communicate without hard-coding changing Pod IP addresses.

---

# 71. Service Accounts

A ServiceAccount gives a Pod an identity inside Kubernetes.

Example:

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: app-sa
```

Pod:

```yaml
serviceAccountName: app-sa
```

This becomes important when applications need to interact with the Kubernetes API.

---

# 72. RBAC

**Role-Based Access Control (RBAC)** controls who can perform which Kubernetes actions.

Important objects:

```text
Role
RoleBinding
ClusterRole
ClusterRoleBinding
```

Example:

```text
User/ServiceAccount
        ↓
RoleBinding
        ↓
Role
        ↓
Permissions
```

---

# 73. Role

A Role defines permissions within a namespace.

Example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: pod-reader

rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

---

# 74. RoleBinding

Connects a Role to a user or ServiceAccount.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: read-pods

subjects:
  - kind: ServiceAccount
    name: app-sa

roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

# 75. Helm

**Helm** is a package manager for Kubernetes.

Instead of maintaining many YAML files manually:

```text
deployment.yaml
service.yaml
configmap.yaml
ingress.yaml
```

you can package them into a Helm Chart.

```text
my-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

# 76. Helm Values

`values.yaml`:

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: alpine

service:
  port: 80
```

Template:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
```

Now configuration can change without rewriting the template.

---

# 77. Common Helm Commands

Search/install a chart:

```bash
helm search repo nginx
```

Install:

```bash
helm install myapp ./my-chart
```

List releases:

```bash
helm list
```

Upgrade:

```bash
helm upgrade myapp ./my-chart
```

Rollback:

```bash
helm rollback myapp 1
```

Uninstall:

```bash
helm uninstall myapp
```

---

# 78. kubectl Basics

Check cluster:

```bash
kubectl cluster-info
```

Get nodes:

```bash
kubectl get nodes
```

Get Pods:

```bash
kubectl get pods
```

All namespaces:

```bash
kubectl get pods -A
```

Detailed information:

```bash
kubectl describe pod <pod-name>
```

---

# 79. Creating Resources

Apply YAML:

```bash
kubectl apply -f deployment.yaml
```

Delete:

```bash
kubectl delete -f deployment.yaml
```

Create a resource imperatively:

```bash
kubectl create deployment nginx --image=nginx
```

---

# 80. Logs

View logs:

```bash
kubectl logs <pod-name>
```

Follow logs:

```bash
kubectl logs -f <pod-name>
```

Specific container:

```bash
kubectl logs <pod-name> -c <container-name>
```

Previous crashed container:

```bash
kubectl logs <pod-name> --previous
```

---

# 81. Execute Commands Inside a Pod

```bash
kubectl exec -it <pod-name> -- sh
```

If Bash exists:

```bash
kubectl exec -it <pod-name> -- bash
```

Run a single command:

```bash
kubectl exec <pod-name> -- ls /app
```

---

# 82. Debugging Kubernetes

When an application isn't working, follow this order:

```text
1. Is the node healthy?
        ↓
2. Is the Pod running?
        ↓
3. Check Pod events
        ↓
4. Check container logs
        ↓
5. Check Service
        ↓
6. Check Endpoints/EndpointSlices
        ↓
7. Check DNS
        ↓
8. Check NetworkPolicy
        ↓
9. Check Ingress/Load Balancer
```

Useful commands:

```bash
kubectl get nodes
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get svc
kubectl get endpointslice
kubectl describe svc <service>
```

---

# 83. CrashLoopBackOff

Meaning:

```text
Container starts
    ↓
Application crashes
    ↓
Container restarts
    ↓
Application crashes
    ↓
Backoff
```

Check:

```bash
kubectl logs <pod>
```

and:

```bash
kubectl describe pod <pod>
```

Common causes:

* Application error
* Missing environment variable
* Incorrect command
* Configuration problem
* Dependency unavailable
* Permission issue

---

# 84. ImagePullBackOff

Means Kubernetes cannot successfully pull the container image.

Possible causes:

* Wrong image name
* Wrong tag
* Private registry
* Missing imagePullSecret
* Registry authentication failure
* Network problems

Check:

```bash
kubectl describe pod <pod>
```

---

# 85. Pending Pod

A Pod in `Pending` state has not successfully started.

Possible reasons:

* Insufficient CPU
* Insufficient memory
* Node selector mismatch
* Affinity rules
* Taints
* PVC problems
* Scheduling constraints

Use:

```bash
kubectl describe pod <pod>
```

Look at the events section.

---

# 86. OOMKilled

`OOMKilled` generally indicates the container exceeded its memory limit or the node experienced memory pressure leading to termination.

Check:

```bash
kubectl describe pod <pod>
```

Then review:

```yaml
resources:
  requests:
    memory: "256Mi"

  limits:
    memory: "512Mi"
```

---

# 87. Rolling Deployment Strategy

A production Deployment can specify update behavior.

Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Concept:

```text
Old version
   ↓
Create new Pod
   ↓
New Pod ready
   ↓
Remove old Pod
   ↓
Repeat
```

---

# 88. Blue-Green Deployment

Two environments:

```text
Blue  → Current production
Green → New version
```

Traffic:

```text
Users
  ↓
Service
  ↓
Blue
```

After validation:

```text
Users
  ↓
Service
  ↓
Green
```

Useful when you want a clear separation between old and new versions.

---

# 89. Canary Deployment

Instead of sending all traffic to the new version:

```text
95% → Old version
5%  → New version
```

Then gradually increase:

```text
90/10
75/25
50/50
0/100
```

Canary deployments often require an ingress controller, Gateway API implementation, service mesh, or other traffic-management mechanism.

---

# 90. Pod Disruption Budget

A **PodDisruptionBudget (PDB)** helps maintain application availability during voluntary disruptions.

Example concept:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: app-pdb

spec:
  minAvailable: 2

  selector:
    matchLabels:
      app: backend
```

This is particularly useful for highly available applications.

---

# 91. SecurityContext

SecurityContext controls security-related settings.

Example:

```yaml
securityContext:
  runAsNonRoot: true
```

Container-level example:

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

Production containers should follow least-privilege principles.

---

# 92. Kubernetes Secrets in Production

Avoid:

```yaml
stringData:
  password: real-production-password
```

inside a public Git repository.

Better approaches include:

* External secret managers
* Cloud secret-management services
* Sealed Secrets
* External Secrets Operator
* CI/CD secret injection
* Kubernetes Secrets with encryption at rest and strict RBAC

---

# 93. Monitoring

A production Kubernetes environment needs observability.

Typical stack:

```text
Kubernetes
    │
    ├── Metrics
    │      ↓
    │   Prometheus
    │      ↓
    │   Grafana
    │
    ├── Logs
    │      ↓
    │   Loki / Elasticsearch
    │
    └── Traces
           ↓
       OpenTelemetry
```

Common metrics:

* CPU
* Memory
* Pod restarts
* Request rate
* Error rate
* Latency
* Node health

---

# 94. Logging

Applications should generally write logs to stdout/stderr.

Example:

```text
Application
     ↓
stdout/stderr
     ↓
Kubernetes logging pipeline
     ↓
Log collector
     ↓
Centralized logging system
```

Common tools include:

* Fluent Bit
* Fluentd
* Loki
* Elasticsearch
* OpenSearch

---

# 95. Kubernetes Production Architecture

A simplified production environment:

```text
                    Internet
                       │
                       ↓
              Load Balancer
                       │
                       ↓
                 Ingress/Gateway
                       │
             ┌─────────┴─────────┐
             ↓                   ↓
        Frontend Service    Backend Service
             │                   │
        ┌────┼────┐         ┌────┼────┐
        ↓    ↓    ↓         ↓    ↓    ↓
       Pod  Pod  Pod        Pod  Pod  Pod
                              │
                              ↓
                         Database
                              │
                              ↓
                          Persistent
                           Storage
```

Supporting systems:

```text
Monitoring
Logging
Secrets
CI/CD
Container Registry
DNS
Backup
Security
```

---

# 96. Kubernetes + Docker + Jenkins CI/CD

A typical DevOps pipeline:

```text
Developer
    ↓
GitHub
    ↓
Jenkins
    ↓
Build
    ↓
Test
    ↓
Docker Build
    ↓
Docker Image
    ↓
Container Registry
    ↓
Kubernetes
    ↓
Deployment
    ↓
Service
    ↓
Users
```

Example:

```text
git push
   ↓
Jenkins Pipeline
   ↓
docker build
   ↓
docker push
   ↓
Update Kubernetes image
   ↓
kubectl apply
   ↓
Rolling Deployment
```

---

# 97. Kubernetes Project Example

A very useful DevOps project is:

## Three-Tier Application

```text
                Internet
                   ↓
                Ingress
                   ↓
              Frontend
                   ↓
               Backend
                   ↓
               Database
```

Components:

```text
Frontend
React
Nginx

Backend
Node.js
Express

Database
MongoDB/PostgreSQL
```

Kubernetes objects:

```text
Namespace
│
├── Frontend Deployment
├── Frontend Service
│
├── Backend Deployment
├── Backend Service
│
├── Database StatefulSet
├── Database Service
│
├── ConfigMap
├── Secret
│
└── Ingress
```

---

# 98. Example Backend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: backend

spec:
  replicas: 3

  selector:
    matchLabels:
      app: backend

  template:
    metadata:
      labels:
        app: backend

    spec:
      containers:
        - name: backend
          image: myregistry/backend:1.0

          ports:
            - containerPort: 3000

          envFrom:
            - configMapRef:
                name: backend-config

            - secretRef:
                name: backend-secret

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"

            limits:
              cpu: "500m"
              memory: "512Mi"

          readinessProbe:
            httpGet:
              path: /health
              port: 3000

          livenessProbe:
            httpGet:
              path: /health
              port: 3000
```

---

# 99. Example Backend Service

```yaml
apiVersion: v1
kind: Service

metadata:
  name: backend-service

spec:
  selector:
    app: backend

  ports:
    - port: 80
      targetPort: 3000

  type: ClusterIP
```

Now another Pod can communicate with:

```text
http://backend-service
```

---

# 100. Kubernetes Commands Cheat Sheet

## Cluster

```bash
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
```

## Pods

```bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl delete pod <pod>
kubectl logs <pod>
kubectl exec -it <pod> -- sh
```

## Deployments

```bash
kubectl get deployments
kubectl describe deployment <deployment>
kubectl scale deployment <deployment> --replicas=5
kubectl rollout status deployment <deployment>
kubectl rollout history deployment <deployment>
kubectl rollout undo deployment <deployment>
```

## Services

```bash
kubectl get services
kubectl describe service <service>
```

## Configuration

```bash
kubectl get configmaps
kubectl get secrets
```

## YAML

```bash
kubectl apply -f file.yaml
kubectl delete -f file.yaml
```

## All resources

```bash
kubectl get all
```

---

# 101. Imperative vs Declarative

### Imperative

You tell Kubernetes what command to execute.

```bash
kubectl create deployment nginx --image=nginx
```

### Declarative

You define desired state in YAML:

```yaml
replicas: 3
```

Then:

```bash
kubectl apply -f deployment.yaml
```

For production infrastructure, declarative configuration is generally preferred because it can be version-controlled and reviewed.

---

# 102. Kubernetes Manifests and Git

A typical repository:

```text
project/
│
├── application/
│
├── Dockerfile
│
├── .dockerignore
│
└── k8s/
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── secret.yaml
    └── ingress.yaml
```

For larger environments:

```text
k8s/
├── base/
└── overlays/
    ├── dev/
    ├── staging/
    └── production/
```

Tools such as **Kustomize** and Helm are commonly used for configuration management.

---

# 103. Kubernetes vs Docker Compose

| Docker Compose                          | Kubernetes                                   |
| --------------------------------------- | -------------------------------------------- |
| Simple multi-container application      | Container orchestration platform             |
| Easy to learn                           | More complex                                 |
| Excellent for local development         | Excellent for production-scale orchestration |
| Usually simpler single-host deployments | Designed for clusters                        |
| Simple scaling                          | Advanced scaling                             |
| Basic networking                        | Advanced networking                          |
| Basic service management                | Advanced workload management                 |
| Smaller operational overhead            | Higher operational complexity                |

Compose can still be appropriate for small production deployments.

---

# 104. Kubernetes vs Virtual Machines

```text
Virtual Machine
    ↓
Guest OS
    ↓
Application
```

Containers:

```text
Container
    ↓
Shared Host Kernel
    ↓
Application
```

Kubernetes:

```text
Kubernetes
    ↓
Container workloads
    ↓
Multiple nodes
```

Kubernetes is not a replacement for the underlying infrastructure. It is an orchestration layer running on nodes.

---

# 105. Important Kubernetes Production Checklist

Before calling an application production-ready, consider:

### Application

```text
☐ Container image is versioned
☐ Image is scanned
☐ Application handles graceful shutdown
☐ Health endpoints exist
☐ Logs go to stdout/stderr
```

### Kubernetes

```text
☐ Deployment configured
☐ Appropriate replica count
☐ Resource requests
☐ Resource limits
☐ Readiness probe
☐ Liveness probe
☐ Startup probe where needed
☐ Rolling update strategy
☐ PodDisruptionBudget where appropriate
```

### Networking

```text
☐ Service configured
☐ Ingress/Gateway configured
☐ TLS configured
☐ Network policies considered
```

### Security

```text
☐ Non-root container
☐ RBAC configured
☐ Secrets protected
☐ Least privilege
☐ Image scanning
☐ SecurityContext configured
```

### Operations

```text
☐ Monitoring
☐ Logging
☐ Alerting
☐ Backups
☐ Disaster recovery
☐ Resource monitoring
```

---

# 106. Most Important Kubernetes Concepts to Memorize

For a DevOps fresher interview, prioritize:

```text
1. Kubernetes architecture
2. Pod
3. Deployment
4. ReplicaSet
5. Service
6. ClusterIP
7. NodePort
8. LoadBalancer
9. Ingress
10. ConfigMap
11. Secret
12. Namespace
13. PV
14. PVC
15. StorageClass
16. StatefulSet
17. DaemonSet
18. Job
19. CronJob
20. Requests & Limits
21. Liveness Probe
22. Readiness Probe
23. Startup Probe
24. HPA
25. RBAC
26. ServiceAccount
27. NetworkPolicy
28. Helm
29. Rolling Update
30. Rollback
31. Taints & Tolerations
32. Node Affinity
33. Troubleshooting
34. Kubernetes + CI/CD
```

---

# 107. The Kubernetes Mental Model

The easiest way to remember Kubernetes is:

```text
                KUBERNETES
                     │
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
    Workloads     Networking     Storage
       │             │             │
       ↓             ↓             ↓
     Pod          Service          PV
     Deployment   Ingress          PVC
     StatefulSet  NetworkPolicy    StorageClass
     DaemonSet
     Job
     CronJob
       │
       ↓
 Configuration
       │
 ┌─────┴─────┐
 ↓           ↓
ConfigMap   Secret
```

Then add:

```text
Security
   ↓
RBAC
ServiceAccount
SecurityContext

Scaling
   ↓
HPA
VPA
Cluster Autoscaler

Operations
   ↓
Monitoring
Logging
Troubleshooting

Package Management
   ↓
Helm
```

---

# 108. Kubernetes Learning Roadmap

For a DevOps engineer, learn Kubernetes in this order:

```text
Stage 1
Docker fundamentals
       ↓
Stage 2
Kubernetes architecture
       ↓
Stage 3
Pods
       ↓
Stage 4
Deployments + ReplicaSets
       ↓
Stage 5
Services + Networking
       ↓
Stage 6
ConfigMaps + Secrets
       ↓
Stage 7
Storage
       ↓
Stage 8
Ingress
       ↓
Stage 9
Probes + Resources
       ↓
Stage 10
Scaling
       ↓
Stage 11
RBAC + Security
       ↓
Stage 12
Helm
       ↓
Stage 13
Troubleshooting
       ↓
Stage 14
CI/CD
       ↓
Stage 15
Production Kubernetes
```

---

# 109. Projects to Become Confident in Kubernetes

## Project 1 — Nginx

Learn:

```text
Pod
Service
Deployment
```

---

## Project 2 — Node.js API

Learn:

```text
Docker
Deployment
Service
ConfigMap
Secret
```

---

## Project 3 — React + Node.js

Learn:

```text
Frontend Deployment
Backend Deployment
Services
Ingress
```

---

## Project 4 — Three-Tier Application

Learn:

```text
Frontend
Backend
Database
Ingress
ConfigMap
Secret
PVC
```

---

## Project 5 — Jenkins + Docker + Kubernetes

Learn:

```text
GitHub
 ↓
Jenkins
 ↓
Docker
 ↓
Registry
 ↓
Kubernetes
```

This is particularly valuable for a DevOps portfolio.

---

## Project 6 — Kubernetes Monitoring

Build:

```text
Kubernetes
    ↓
Prometheus
    ↓
Grafana
```

Monitor:

```text
CPU
Memory
Pods
Nodes
Restarts
Application metrics
```

---

## Project 7 — Production-Style Deployment

Include:

```text
Ingress
TLS
Helm
HPA
RBAC
NetworkPolicy
Resource limits
Health probes
Monitoring
Logging
CI/CD
```

This project will give you much stronger practical Kubernetes knowledge than simply creating individual YAML files.

---

# 110. Final Interview Summary

If an interviewer asks:

### "What is Kubernetes?"

Answer:

> Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, networking, and management of containerized applications across a cluster of machines.

### "What is a Pod?"

> A Pod is the smallest deployable unit in Kubernetes and can contain one or more containers that share networking and storage.

### "Deployment vs Pod?"

```text
Pod
 ↓
Runs application containers

Deployment
 ↓
Manages Pods
 ↓
Scaling
 ↓
Rolling updates
 ↓
Rollback
```

### "Service vs Pod?"

```text
Pod
 ↓
Ephemeral IP

Service
 ↓
Stable network endpoint
 ↓
Routes traffic to Pods
```

### "ConfigMap vs Secret?"

```text
ConfigMap → Non-sensitive configuration
Secret    → Sensitive configuration
```

### "HPA?"

> Horizontal Pod Autoscaler automatically adjusts the number of Pod replicas based on configured metrics.

### "PV vs PVC?"

```text
PV  → Storage resource
PVC → Request for storage
```

### "Deployment vs StatefulSet?"

```text
Deployment  → Stateless workloads
StatefulSet → Stateful workloads requiring stable identity/storage
```

### "Liveness vs Readiness?"

```text
Liveness  → Should the container be restarted?
Readiness → Should the Pod receive traffic?
```

### "Why Helm?"

> Helm packages and templates Kubernetes resources, making it easier to deploy, configure, version, and manage complex applications.

---

# 111. One-Page Kubernetes Architecture

```text
                         USERS
                           │
                           ↓
                  Load Balancer / DNS
                           │
                           ↓
                    Ingress / Gateway
                           │
                           ↓
                       SERVICE
                           │
              ┌────────────┼────────────┐
              ↓            ↓            ↓
             POD          POD          POD
              │            │            │
          Container    Container    Container
              │            │            │
              └────────────┼────────────┘
                           ↓
                     APPLICATION


                  KUBERNETES CLUSTER
                  ──────────────────

                     CONTROL PLANE
                           │
       ┌───────────────────┼───────────────────┐
       ↓                   ↓                   ↓
  API Server           Scheduler        Controller Manager
       │
       ↓
      etcd


                     WORKER NODES
                           │
       ┌───────────────────┼───────────────────┐
       ↓                   ↓                   ↓
     Node 1              Node 2              Node 3
       │                   │                   │
    kubelet             kubelet             kubelet
       │                   │                   │
     Pods                Pods                Pods


              SUPPORTING COMPONENTS
              ──────────────────────

    ConfigMap       Secret          RBAC
    Storage         HPA             NetworkPolicy
    Helm            Monitoring      Logging
    Ingress         CI/CD           Registry
```

---

# 112. Final DevOps Mental Model

The complete flow to remember is:

```text
Developer
    │
    ↓
GitHub
    │
    ↓
Jenkins / CI
    │
    ├── Test
    │
    ├── Build
    │
    └── Docker Build
            │
            ↓
      Docker Image
            │
            ↓
    Container Registry
            │
            ↓
       Kubernetes
            │
       ┌────┴────┐
       ↓         ↓
 Deployment    Service
       │         │
       ↓         ↓
     Pods ←──────┘
       │
       ├── ConfigMap
       ├── Secret
       ├── Storage
       ├── Probes
       ├── Resources
       └── Security
            │
            ↓
      Ingress / Gateway
            │
            ↓
          Users

      ┌──────────────────┐
      │   Monitoring     │
      │   Prometheus     │
      │   Grafana        │
      │   Logging        │
      └──────────────────┘
```

**Core principle:** Docker packages the application into containers; Kubernetes manages those containers at scale; CI/CD delivers new versions; networking exposes them; storage preserves state; security controls access; and monitoring tells you what is happening.
