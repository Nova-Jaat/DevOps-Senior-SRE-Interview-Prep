# DevOps Engineer 1st Round Interview: Top 20 Questions & Master Q&A Guide

A comprehensive, production-grade guide covering the top 20 questions asked in first-round DevOps and Site Reliability Engineering (SRE) technical interviews. Each answer is structured with **Core Answer**, **Under the Hood / Deep Dive**, **Diagnostic Commands & Code Examples**, and **What Interviewers Are Listening For**.

---

## Table of Contents
1. [End-to-End CI/CD Pipeline (Commit to Production)](#1-end-to-end-cicd-pipeline-commit-to-production)
2. [What Happens Internally When You Run a Docker Container](#2-what-happens-internally-when-you-run-a-docker-container)
3. [How Kubernetes Schedules Pods on Nodes](#3-how-kubernetes-schedules-pods-on-nodes)
4. [CrashLoopBackOff: Root Causes and Debugging](#4-crashloopbackoff-root-causes-and-debugging)
5. [Difference Between Readiness, Liveness, and Startup Probes](#5-difference-between-readiness-liveness-and-startup-probes)
6. [Kubernetes Architecture: Control Plane & Node Internals](#6-kubernetes-architecture-control-plane--node-internals)
7. [Service Discovery and Traffic Routing in Kubernetes](#7-service-discovery-and-traffic-routing-in-kubernetes)
8. [What Happens Internally When You Run `kubectl apply`](#8-what-happens-internally-when-you-run-kubectl-apply)
9. [Deployment vs StatefulSet vs DaemonSet](#9-deployment-vs-statefulset-vs-daemonset)
10. [Configuration Management Across Environments](#10-configuration-management-across-environments)
11. [How Terraform Manages State and Dependencies Internally](#11-how-terraform-manages-state-and-dependencies-internally)
12. [Terraform Drift: Detection and Remediation](#12-terraform-drift-detection-and-remediation)
13. [Designing Infrastructure as Code for Multi-Environment Environments](#13-designing-infrastructure-as-code-for-multi-environment-environments)
14. [DNS Resolution Step-by-Step](#14-dns-resolution-step-by-step)
15. [Layer 4 vs Layer 7 Load Balancing](#15-layer-4-vs-layer-7-load-balancing)
16. [Identifying Application Bottlenecks Under Load](#16-identifying-application-bottlenecks-under-load)
17. [Designing Production Monitoring, Observability, and Alerting](#17-designing-production-monitoring-observability-and-alerting)
18. [Secure Service-to-Service Authentication (mTLS & Zero Trust)](#18-secure-service-to-service-authentication-mtls--zero-trust)
19. [Zero-Downtime Deployment Strategies](#19-zero-downtime-deployment-strategies)
20. [Production Outage: Step-by-Step Incident Debugging Framework](#20-production-outage-step-by-step-incident-debugging-framework)

---

## 1. End-to-End CI/CD Pipeline (Commit to Production)

### Core Answer
A production-grade CI/CD pipeline automates code integration, testing, packaging, security validation, and deployment using a GitOps pull-based or event-driven model. It guarantees that every artifact promoted to production is immutable, validated, cryptographically signed, and deployed without downtime.

### End-to-End Workflow & Deep Dive
```
[Developer Git Push]
         │ (Webhook trigger)
         ▼
[Phase 1: Continuous Integration (CI Runner)]
├── Linting & Formatting (flake8, eslint, golangci-lint)
├── Unit Tests & Code Coverage (pytest, go test, jest)
├── Static Application Security Testing (SAST: SonarQube, Semgrep)
└── Software Composition Analysis (SCA: Snyk, Dependabot, Trivy)
         │ (Pass)
         ▼
[Phase 2: Build & Artifact Generation]
├── Multi-stage Docker build (minimal distroless / alpine image)
├── Container Vulnerability Scan (Trivy, Clair, AWS Inspector)
├── Image Signing (Cosign / Sigstore)
└── Push to OCI Registry (ECR, Harbor) with immutable tag (Git SHA-1234abc)
         │
         ▼
[Phase 3: Continuous Delivery / GitOps Bridge]
├── Automated PR updates environment manifest repo (Kustomize/Helm image tag)
└── GitOps Controller (ArgoCD / FluxCD) detects state drift
         │
         ▼
[Phase 4: Progressive Deployment (Cluster)]
├── Staging: Automated integration/smoke tests
└── Production: Canary / Blue-Green rollout via Argo Rollouts
    ├── Initial traffic split (e.g., 10% canary)
    ├── Automated metric analysis (Prometheus error rate, p99 latency)
    └── Automated promotion to 100% or instant automated rollback
```

### Key Production Best Practices
- **Never use mutable tags like `:latest`**: Always tag images with the short Git commit SHA (`v1.4.2-a8f3b21`).
- **Separation of CI and CD**: CI builds the image and tests; CD pulls configuration from a dedicated GitOps repository. The CI runner has no direct cluster write credentials.
- **Fail Fast**: Linting and unit tests run in parallel within the first 90 seconds.

---

## 2. What Happens Internally When You Run a Docker Container?

### Core Answer
Docker is a high-level wrapper around Linux kernel primitives. Running `docker run -d -p 80:80 nginx` initiates an API call from the Docker CLI to `dockerd`, which delegates to `containerd`, which invokes `runc`. `runc` interacts with the Linux kernel to create isolated namespaces, set cgroup resource bounds, configure an OverlayFS root filesystem, and execute the entrypoint process.

### Deep Dive: Step-by-Step Kernel Flow
```
docker CLI ──(UNIX Socket)──> dockerd ──(gRPC)──> containerd ──(fork/exec)──> runc ──(syscalls)──> Linux Kernel
                                                       │
                                                       └──> containerd-shim (supervises container)
```

1. **Client to Engine**: The CLI translates the command into a REST API call over `/var/run/docker.sock` to the `dockerd` daemon.
2. **Containerd & Image Unpack**: `dockerd` hands off execution to `containerd` via gRPC. `containerd` checks local storage or pulls the OCI image from the registry.
3. **Storage Setup (Overlay2)**:
   - Layers in the image are mounted as read-only directories (`lowerdir`).
   - A new writable directory is mounted on top (`upperdir`).
   - A `workdir` handles copy-on-write actions.
   - The unified view is exposed at `merged/`.
4. **Invocation of `runc`**: `containerd` executes `runc` passing an OCI `config.json` specification.
5. **Kernel Primitives Applied**:
   - **Linux Namespaces (`clone()` / `unshare()` / `setns()` flags)**:
     - `CLONE_NEWPID`: Process isolation (app becomes PID 1 inside container).
     - `CLONE_NEWNET`: Dedicated network stack (loopback, virtual ethernet `veth` pair bridging to `docker0`).
     - `CLONE_NEWNS`: Independent mount table (`pivot_root` changes root to the Overlay2 merged directory).
     - `CLONE_NEWIPC`: Isolated System V IPC and POSIX message queues.
     - `CLONE_NEWUTS`: Isolated hostname and domain name.
     - `CLONE_NEWUSER`: UID/GID mapping (maps container root to non-privileged host UID).
   - **Control Groups (`cgroups v2` in `/sys/fs/cgroup`)**:
     - Constrains CPU shares/quotas (`cpu.max`), memory limits (`memory.max`), and max process threads (`pids.max`).
   - **Security Filters**:
     - **Seccomp**: Drops dangerous syscalls (e.g., `sys_chroot`, `ptrace`, `reboot`).
     - **Capabilities**: Drops default root powers (e.g., drops `CAP_SYS_ADMIN`, `CAP_NET_RAW`).
     - **AppArmor / SELinux**: Enforces MAC (Mandatory Access Control) policies.
6. **Process Handoff**: `runc` calls `execve()` to replace itself with the application binary. `runc` exits, leaving `containerd-shim` as the subreaper parent process to collect exit codes and stdio streams without keeping the runtime daemon attached.

---

## 3. How Kubernetes Schedules Pods on Nodes

### Core Answer
The `kube-scheduler` assigns unscheduled pods (`spec.nodeName` is empty) to healthy, optimal worker nodes. It runs a two-stage scheduling cycle: **Filtering (Predicates)** to eliminate ineligible nodes, and **Scoring (Priorities)** to rank the remaining nodes and pick the highest scorer.

### Deep Dive: The Scheduling Pipeline
```
[Unscheduled Pod Detected via Informer]
                 │
                 ▼
     [1. Pre-Filter / Filtering Phase]
     ├── NodeResourcesFit (CPU/Memory requests vs Allocatable capacity)
     ├── NodeName (exact match if specified)
     ├── NodePorts (checks for host port collisions)
     ├── PodTopologySpread (spreads across AZs / failure-domains)
     ├── NodeSelector & NodeAffinity (requiredDuringSchedulingIgnoredDuringExecution)
     └── Taints and Tolerations (rejects nodes tainted unless pod tolerates them)
                 │
                 ▼ (Eligible Candidates)
     [2. Scoring Phase (Score: 0 - 100)]
     ├── NodeResourcesBalancedAllocation (avoids resource fragmentation)
     ├── ImageLocalityPriority (favors node with image already pulled)
     ├── PodAffinity / PodAntiAffinity (preferredDuringScheduling)
     └── NodeAffinity (preferred weights)
                 │
                 ▼
     [3. Reserve & Permit Phase]
     └── Reserves resources in local scheduler cache; executes permit plugins
                 │
                 ▼
     [4. Binding Phase]
     └── POST to API Server: creates a Binding object (spec.nodeName = selected-node)
```

### What Happens Next?
The `kubelet` running on the selected node watches the API server. When it sees a pod bound to its own node name, it invokes the CRI (containerd) to pull images and run the container, and calls the CNI plugin to configure pod networking.

---

## 4. CrashLoopBackOff: Root Causes and Debugging

### Core Answer
`CrashLoopBackOff` is a Kubernetes pod status indicating that a container started, crashed (exited with non-zero exit code or was terminated), and kubelet restarted it repeatedly, backing off exponentially between restarts (10s, 20s, 40s, up to 5 minutes) to protect node stability.

### Common Root Causes
| Exit Code | Cause | Common Reason |
|---|---|---|
| **137** | `SIGKILL` (OOMKilled) | Container exceeded its memory limit or was killed by Linux host OOM-killer. |
| **139** | `SIGSEGV` | Segmentation fault in binary / corrupted shared library. |
| **143** | `SIGTERM` | Graceful termination signal sent (timed out or failed liveness probe). |
| **1** | Application Error | Unhandled exception, missing DB connection string, unmounted ConfigMap/Secret. |
| **127** | Command Not Found | Wrong entrypoint or binary missing in container path. |

### Systematic Debugging Steps
```bash
# 1. Identify the failing pod and restart count
kubectl get pods -n <namespace> -o wide

# 2. Inspect exit code, termination reason, and recent Events
kubectl describe pod <pod-name> -n <namespace>
# Look under: "Last State: Terminated" -> Reason (e.g. OOMKilled) & Exit Code

# 3. View logs of the CRASHED instance (critical step)
kubectl logs <pod-name> -n <namespace> --previous

# 4. If multi-container pod:
kubectl logs <pod-name> -c <container-name> -n <namespace> --previous

# 5. If it crashes immediately before flushing logs, override entrypoint for interactive triage:
kubectl debug pod/<pod-name> -it --image=busybox --target=<container-name>
```

---

## 5. Difference Between Readiness, Liveness, and Startup Probes

### Core Comparison
| Feature | Liveness Probe | Readiness Probe | Startup Probe |
|---|---|---|---|
| **Purpose** | Detects deadlocks, freezes, or crashed internal loops. | Detects whether the app is ready to accept external network traffic. | Protects slow-starting legacy apps during bootstrap. |
| **Action on Failure** | Kubelet **kills** the container and restarts it. | Pod IP is removed from Service **Endpoints/EndpointSlices**. Container is NOT restarted. | Kubelet kills and restarts container; disables liveness/readiness until passed. |
| **Best Practice** | Check lightweight, local process health (e.g., `/healthz`). | Check local caches/readiness (e.g., `/ready`). | Use for apps taking >30s to boot (initial cache warmup). |
| **Critical Anti-Pattern** | Checking external DB/Redis. If DB slows down, ALL pods get restarted simultaneously, causing a total cascade! | Blocking on non-critical third-party external services. | Setting low `failureThreshold` on startup probe. |

### Production Manifest Example
```yaml
spec:
  containers:
  - name: backend-api
    image: backend-api:v1.2.0
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10 # Gives up to 300s to boot
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 15
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 2
```

---

## 6. Kubernetes Architecture: Control Plane & Node Internals

### Control Plane Components
1. **`kube-apiserver`**:
   - The central REST API gateway and front door of the cluster.
   - Handles Authentication (AuthN), Authorization (RBAC), Admission Control (Mutating and Validating Webhooks), and schema validation.
   - The **only** component that directly reads and writes to `etcd`.
   - Completely stateless; scales horizontally behind a load balancer.
2. **`etcd`**:
   - High-performance, distributed, strongly consistent Key-Value store implementing the Raft consensus algorithm.
   - Stores complete cluster state, specs, and metadata.
3. **`kube-scheduler`**:
   - Watches for unscheduled pods and assigns them to nodes via filtering and scoring.
4. **`kube-controller-manager`**:
   - Runs background reconciliation control loops: `DeploymentController`, `ReplicaSetController`, `NodeController`, `EndpointSliceController`, `NamespaceController`.
   - Constantly drives current cluster state toward desired state.
5. **`cloud-controller-manager`**:
   - Integrates with cloud provider APIs (provisions AWS ALBs, Azure Load Balancers, EBS storage, cloud node lifecycles).

### Worker Node Components
1. **`kubelet`**:
   - Primary node agent. Registers node with API server, watches PodSpecs assigned to the node, instructs container runtime via CRI, manages volume mounts via CSI, executes probes, and reports node and pod status back to API server.
2. **`kube-proxy`**:
   - Manages network rules on each node (using `iptables` or `IPVS`) to translate virtual Service ClusterIPs to Pod IPs. (In eBPF clusters like Cilium, kube-proxy is replaced).
3. **Container Runtime (CRI)**:
   - Low-level engine (containerd, CRI-O) that pulls images and delegates execution to OCI runtimes (`runc`).
4. **CNI (Container Network Interface)**:
   - Sets up network namespace, IPAM, and pod routes (Calico, Cilium, AWS VPC CNI).

---

## 7. Service Discovery and Traffic Routing in Kubernetes

### Core Answer
Service discovery in Kubernetes connects clients to dynamic, ephemeral pod IPs using **CoreDNS**, **Kubernetes Services**, and **kube-proxy / eBPF rules**.

### Step-by-Step Resolution & Routing
```
Client Pod
   │ 1. DNS Query: "payment-service"
   ▼
CoreDNS (nameserver 10.96.0.10)
   │ 2. Resolves to virtual ClusterIP: 10.100.50.25
   ▼
Kernel Packet Translation (iptables / IPVS / eBPF)
   │ 3. DNAT rewrites 10.100.50.25:80 -> healthy Pod IP 10.244.2.14:8080
   ▼
Target Backend Pod
```

1. **DNS Resolution**:
   - Pod's `/etc/resolv.conf` points to CoreDNS (`nameserver 10.96.0.10`) with search domains:
     `default.svc.cluster.local`, `svc.cluster.local`, `cluster.local`.
   - Querying `payment-service` resolves to `payment-service.default.svc.cluster.local` -> returns virtual `ClusterIP`.
2. **Endpoint Tracking**:
   - The `EndpointSliceController` monitors pods matching the Service's `selector`.
   - Only pods passing **Readiness Probes** are added to the `EndpointSlice` object.
3. **Data Plane Packet Routing**:
   - The ClusterIP is a purely virtual IP (not assigned to any physical network card).
   - `kube-proxy` programs Linux kernel rules:
     - **iptables mode**: Uses `PREROUTING` rules with `statistic mode random` to probabilistically distribute packets across pod IPs.
     - **IPVS mode**: Uses Linux IP Virtual Server kernel hash tables for $O(1)$ lookups and round-robin scheduling (much faster at >5,000 services).
     - **Cilium eBPF mode**: Replaces kube-proxy; intercepts socket connections directly (`sock_ops`) to execute translation without traversing iptables chains.
4. **Headless Services (`clusterIP: None`)**:
   - Returns direct `A` records of pod IPs instead of a ClusterIP, allowing stateful clients (e.g. Kafka, Cassandra) to perform client-side load balancing.

---

## 8. What Happens When You Run `kubectl apply` Internally?

### Step-by-Step Internal Execution Flow
```
kubectl apply -f app.yaml
        │
        ├── [Client Side]
        │   ├── Scans OpenAPI schema for client-side validation
        │   └── Computes Three-Way Merge Patch (last-applied-config, live state, local file)
        │
        ▼ (HTTPS REST / PATCH)
kube-apiserver
        │
        ├── 1. Authentication (Verifies X.509 cert / OIDC bearer token)
        ├── 2. Authorization (RBAC: checks Role/ClusterRole verbs & resources)
        ├── 3. Mutating Admission Controllers (e.g. injects sidecar, sets defaults)
        ├── 4. Schema Validation (Strict JSON schema validation)
        ├── 5. Validating Admission Controllers (OPA Gatekeeper, Kyverno policies)
        └── 6. Persistence: Commits JSON object to etcd
                 │
                 ▼ (Watch Event Triggered)
DeploymentController
        └── Compares desired replicas with live state -> creates/updates ReplicaSet
                 │
                 ▼ (Watch Event Triggered)
ReplicaSetController
        └── Creates Pod objects with spec.nodeName = ""
                 │
                 ▼ (Watch Event Triggered)
kube-scheduler
        └── Filters & Scores nodes -> Writes Binding object (spec.nodeName = "node-1")
                 │
                 ▼ (Watch Event Triggered)
kubelet (on node-1)
        ├── Instructs CRI (containerd) to pull image & create container
        ├── Instructs CNI to create veth pair & assign IP
        ├── Instructs CSI to attach & mount persistent volumes
        └── Starts container and reports Ready status back to apiserver
```

---

## 9. Deployment vs StatefulSet vs DaemonSet

### Comparison Matrix
| Attribute | Deployment | StatefulSet | DaemonSet |
|---|---|---|---|
| **Primary Use Case** | Stateless services (APIs, web servers, microservices). | Stateful services (Databases, Kafka, Redis, Elasticsearch). | Node-level system agents (log collectors, monitoring, CNI). |
| **Pod Identity** | Interchangeable, random hashes (`web-67d7f8c9b-abcde`). | Stable, deterministic ordinals (`kafka-0`, `kafka-1`, `kafka-2`). | One pod per node, tied to node lifecycle (`fluentbit-node-1`). |
| **Storage Handling** | Ephemeral or shared read-write volumes (`ReadWriteMany`). | Unique, dedicated persistent volume per ordinal via `volumeClaimTemplates`. | HostPath or direct node storage mounts. |
| **Scaling Order** | Parallel, non-deterministic. | Strictly sequential (0 $\rightarrow$ N-1 up; N-1 $\rightarrow$ 0 down). | Automatic when nodes join or leave cluster. |
| **Network Identity** | Shared Service ClusterIP. | Dedicated stable DNS records per pod via Headless Service (`pod-0.service.ns.svc`). | Host network or pod network per node. |

---

## 10. Configuration Management Across Environments

### Core Strategy
Follow **12-Factor App** principles: **Strict separation of code and configuration**. The container image artifact is built once and remains 100% immutable across Dev, Staging, and Production; environment variations are injected at runtime.

### Production Implementation Architecture
```
                        [Base Manifests (Kustomize / Helm)]
                                       │
                ┌──────────────────────┼──────────────────────┐
                ▼                      ▼                      ▼
         [Overlay: Dev]        [Overlay: Staging]       [Overlay: Prod]
         - replicas: 1          - replicas: 3            - replicas: 10
         - cpu: 250m            - cpu: 500m              - cpu: 2000m
         - host: dev.api.com    - host: stg.api.com      - host: api.com
```

### Key Components
1. **Kustomize Overlays (DRY Principle)**:
   - `base/`: Common manifests (Deployment, Service, ServiceAccount).
   - `overlays/prod/`: Environment patches modifying replicas, resource requests, ingress hosts, and environment variables without template duplication.
2. **Secrets Management (Never Plaintext in Git)**:
   - **External Secrets Operator (ESO)**: Connects Kubernetes to AWS Secrets Manager, HashiCorp Vault, or Azure Key Vault. Syncs secrets directly into native K8s Secrets in-cluster.
   - **Sealed Secrets / SOPS**: Encrypts secret values with asymmetric keys so encrypted manifests can safely reside in public/private Git repositories.
3. **Automated CI Validation**:
   - `kubeconform` / `kube-linter` validates syntax in CI before Git merge.
   - Conftest / Kyverno validates corporate policies (e.g. disallowing root users, enforcing CPU/Memory limits).

---

## 11. How Terraform Manages State and Dependencies Internally

### 1. State Management (`terraform.tfstate`)
- **Metadata Mapping**: Maps declared HCL resource declarations (`aws_s3_bucket.data`) to physical cloud provider IDs (`arn:aws:s3:::my-company-prod-data`).
- **Performance Cache**: Caches the live infrastructure attributes to avoid querying hundreds of cloud APIs on every operation.
- **Concurrency Locking**: State locking via remote backends (e.g. AWS S3 + DynamoDB table lock, or GCS object locking) prevents race conditions and corrupted states caused by concurrent pipeline runs.

### 2. Dependency Graph (DAG)
Terraform builds an in-memory **Directed Acyclic Graph (DAG)** to model dependencies:
- **Implicit Dependencies**: Detected when an attribute of one resource references another:
  ```hcl
  resource "aws_instance" "web" {
    subnet_id = aws_subnet.public.id # Implicit dependency: subnet must be created first!
  }
  ```
- **Explicit Dependencies**: Declared via `depends_on = [aws_iam_role_policy_attachment.worker]`.
- **Topological Sorting**: Terraform walks the graph to determine which resources can be created in parallel (default `--parallelism=10`) and which must wait.
- **Reverse Destruction**: On destroy or replacement, Terraform traverses the DAG in exact reverse order to prevent orphaned resources.

---

## 12. Terraform Drift: Detection and Remediation

### Core Answer
**Drift** occurs when real-world cloud infrastructure diverges from the state tracked in Terraform state files and declared in HCL code (typically caused by emergency ClickOps in cloud consoles, rogue scripts, or automatic cloud updates).

### Detection Methods
```bash
# 1. Refresh-only inspection (reads live cloud state without proposing changes)
terraform plan -refresh-only

# 2. Standard plan comparison (shows drift and proposed actions to fix it)
terraform plan -detailed-exitcode
# Exit codes: 0 = No changes, 1 = Error, 2 = Succeeded with drift/changes detected
```

### Remediation Strategies
1. **Revert Unauthorized Changes (Enforce Git as Source of Truth)**:
   - Run `terraform apply`. Terraform will overwrite the manual changes and restore infrastructure to the declared HCL code.
2. **Accept Legitimate Emergency Changes**:
   - Update the local HCL code to match the manual change.
   - Run `terraform apply -refresh-only` or `terraform apply` to sync state without destroying the resource.
3. **Continuous Drift Automation**:
   - Run scheduled nightly CI pipelines (e.g. GitHub Actions / Atlantis / Spacelift) executing `terraform plan -detailed-exitcode`.
   - Route exit code 2 alerts to Slack or PagerDuty.

---

## 13. Designing Infrastructure as Code for Multi-Environment Environments

### Recommended Directory Structure (Directory Segregation over Workspaces)
> [!IMPORTANT]
> Avoid using Terraform CLI workspaces (`terraform workspace select prod`) for multi-account environments. Workspaces share backend configurations and risk accidental cross-environment destruction. Use explicit directory segregation or Terragrunt.

```
terraform-infra/
├── modules/                   # Reusable, versioned child modules
│   ├── vpc/
│   ├── eks/
│   └── rds/
└── environments/
    ├── dev/
    │   ├── backend.tf         # S3 Bucket in Dev AWS Account
    │   ├── main.tf            # Calls modules with dev sizing
    │   └── terraform.tfvars
    ├── staging/
    │   ├── backend.tf
    │   └── main.tf
    └── prod/
        ├── backend.tf         # S3 Bucket in Prod AWS Account (Isolated!)
        ├── main.tf            # Multi-AZ, High-Availability, Protected deletion
        └── terraform.tfvars
```

### Key Architectural Tenets
- **Blast Radius Reduction**: Dev, Staging, and Prod live in completely isolated cloud accounts / subscriptions.
- **Module Versioning**: Pin module tags using Git tags (`source = "git::https://github.com/...//vpc?ref=v2.1.0"`).
- **Terragrunt for DRY Architecture**: Eliminates boilerplate backend and provider blocks, keeping configs minimal and DRY across environments.
- **Principle of Least Privilege**: CI runners deploying to Dev have zero IAM credentials to access Staging or Production.

---

## 14. DNS Resolution Step-by-Step

### Core Answer
When a client navigates to `https://api.example.com`, DNS translates the human-friendly FQDN into a routable IP address through a hierarchical query chain.

```
Client Browser
     │ 1. Checks Browser Cache -> OS Cache -> /etc/hosts
     ▼
Recursive DNS Resolver (8.8.8.8 / ISP / VPC Resolver)
     │ 2. Queries Root Server (".")
     ▼
Root Nameserver
     │ 3. Returns Referral to ".com" TLD Server
     ▼
TLD Nameserver (.com)
     │ 4. Returns Referral to Authoritative Server (NS records for example.com)
     ▼
Authoritative Nameserver (e.g., AWS Route 53 / Cloudflare)
     │ 5. Returns A / AAAA Record (e.g., 203.0.113.10) with TTL
     ▼
Recursive Resolver caches record and returns IP to Client OS
     │
Client initiates TCP Handshake (SYN -> SYN-ACK -> ACK) and TLS negotiation
```

### Diagnostic Tools
```bash
# Query full hierarchical trace step by step
dig +trace api.example.com

# Query specific nameserver directly
dig @8.8.8.8 api.example.com +noall +answer
```

---

## 15. Layer 4 vs Layer 7 Load Balancing

### Deep Comparison
| Feature | Layer 4 (Transport: TCP / UDP) | Layer 7 (Application: HTTP / HTTPS / gRPC) |
|---|---|---|
| **Protocol Awareness** | Aware only of IP, Port, TCP/UDP flags. Blind to HTTP payload. | Inspects full HTTP/HTTPS/gRPC request (headers, path, cookies, query parameters). |
| **TLS / SSL Termination** | Pass-through or terminated at transport layer without decrypting headers. | Fully terminates TLS, decrypts payload, inspects/modifies headers, re-encrypts. |
| **Routing Capability** | IP and Port hash only (e.g. Round Robin, Least Connections). | Path-based (`/api` vs `/static`), host-based (`auth.domain.com`), cookie stickiness. |
| **Performance & Latency** | Ultra-high throughput, sub-millisecond latency (millions of req/sec). | Higher CPU/memory overhead due to packet reassembly, TLS decryption, and header parsing. |
| **Cloud Examples** | AWS Network Load Balancer (NLB), HAProxy (mode tcp), IPVS. | AWS Application Load Balancer (ALB), NGINX, Envoy, Traefik. |
| **Best Used For** | Databases, MQTT, gaming servers, raw TCP/UDP, high-throughput gRPC. | REST microservices, web apps, API gateways, WAF integration. |

---

## 16. Identifying Application Bottlenecks Under Load

### Structured Diagnostic Methodology
Follow the **USE Method** (Utilization, Saturation, Errors) for hardware resources and the **RED Method** (Rate, Errors, Duration) for service metrics.

```
[Step 1: Edge & Ingress (RED Method)]
├── Check ALB/Ingress: Spikes in 5xx error rate?
└── Check p95/p99 Latency vs Request Rate (Requests/Sec)
         │
         ▼
[Step 2: Distributed Tracing & APM]
├── OpenTelemetry / Datadog / Jaeger trace inspection
└── Identify slow spans: Is time spent in Application Code, DB queries, or External APIs?
         │
         ▼
[Step 3: Host & Container Level (USE Method)]
├── CPU CFS Throttling: Check container_cpu_cfs_throttled_periods_total
├── Memory & GC: Frequent JVM / Go Garbage Collection pauses (Stop-The-World)?
├── Network: Socket drops, SYN backlog saturation, ephemeral port exhaustion
└── Disk I/O: High iowait, EBS bursting credits depleted
         │
         ▼
[Step 4: Database & Downstream Tier]
├── Active vs Max Connection Pool saturation
├── Slow queries, missing indexes, table lock contention
└── Read replica replication lag
```

### Key Diagnostic Commands
```bash
# Check CPU throttling on Linux / containers
cat /sys/fs/cgroup/cpu/cpu.stat

# Check socket backlogs and connection states
ss -s
netstat -s | grep -i "listen overflows"

# Continuous profiling (on-CPU flame graphs)
pyroscope / perf top
```

---

## 17. Designing Production Monitoring, Observability, and Alerting

### 1. The Four Golden Signals (Google SRE)
1. **Latency**: Time to service a request (measured in percentiles: p50, p95, p99; averages are misleading).
2. **Traffic**: Demand on the system (HTTP requests per second, I/O bandwidth).
3. **Errors**: Rate of failed requests (HTTP 5xx, uncaught exceptions).
4. **Saturation**: How full the service/resource is (CPU quota %, memory usage, DB connection pool utilization).

### 2. The Observability Stack (MELT)
- **Metrics**: Prometheus / Mimir / VictoriaMetrics for numeric time-series data.
- **Events & Alerts**: Alertmanager routing to PagerDuty/Opsgenie.
- **Logs**: Structured JSON logs shipped via Fluentbit/Vector to OpenSearch/Loki.
- **Traces**: OpenTelemetry SDKs exporting distributed spans to Jaeger/Tempo.

### 3. Alerting Philosophy: Eliminating Alert Fatigue
- **Alert on Symptoms (User Impact / SLOs), NOT Causes**: Page the on-call engineer if API error rate breaches 1% over 5 minutes (user-impacting). Do **not** wake anyone up because an individual pod's CPU reached 85%.
- **Multi-Window Multi-Burn-Rate Alerts**: Fire urgent pages on fast budget burn (e.g. 2% error budget consumed in 1 hour); file Jira tickets on slow burns (consuming budget over 3 days).
- **Mandatory Runbook Links**: Every alert must include a direct URL to an actionable runbook detailing immediate mitigation and triage steps.

---

## 18. Secure Service-to-Service Authentication (mTLS & Zero Trust)

### Core Answer
In modern zero-trust cloud architectures, private networks are treated as hostile. Services must authenticate and authorize every request using cryptographic identity rather than relying on network perimeter firewalls.

### Three Pillars of Inter-Service Security
```
[Service A Pod] ──(mTLS encrypted + SPIFFE ID)──> [Service B Pod]
      │                                                   │
      └── Kubernetes ServiceAccount                       └── Validates caller SPIFFE ID
          exchanged for AWS STS token                         & applies RBAC/AuthZ policy
          via OIDC (IRSA / Workload Identity)
```

1. **Mutual TLS (mTLS) with Service Mesh (Istio / Linkerd / Cilium)**:
   - Sidecar proxies (Envoy) automatically inject and rotate short-lived X.509 certificates.
   - Traffic is encrypted in-transit; client authenticates server, and server authenticates client.
2. **Cryptographic Identity via SPIFFE / SPIRE**:
   - Each workload receives a cryptographically verifiable SPIFFE ID:
     `spiffe://cluster.local/ns/prod/sa/payment-service`
   - Services evaluate caller identity inside the mesh AuthorizationPolicy:
     ```yaml
     apiVersion: security.istio.io/v1beta1
     kind: AuthorizationPolicy
     spec:
       rules:
       - from:
         - source:
             principals: ["cluster.local/ns/prod/sa/checkout-service"]
         to:
         - operation:
             methods: ["POST"]
             paths: ["/charge"]
     ```
3. **Cloud IAM Workload Identity (AWS IRSA / GCP Workload Identity)**:
   - Eliminates hardcoded long-lived API keys.
   - Kubernetes ServiceAccount tokens are exchanged via OIDC for short-lived temporary cloud STS credentials.

---

## 19. Zero-Downtime Deployment Strategies

### 1. Rolling Updates (Kubernetes Default)
- Replaces old pods with new pods incrementally.
- Configuration:
  ```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%         # Can create 25% extra pods during rollout
      maxUnavailable: 0     # Never terminate an old pod until a new pod is Healthy
  ```

### 2. Blue-Green Deployment
- Two identical environments: **Blue** (current live) and **Green** (new version).
- Deployment is tested in isolation on Green.
- Traffic router / Load balancer switches 100% of traffic to Green instantaneously.
- Instant rollback: switch router back to Blue if errors appear.

### 3. Canary Deployment (Progressive Delivery via Argo Rollouts / Flagger)
- Routes a tiny fraction of real traffic (e.g., 5%) to the Canary pod.
- Compares real-world metrics (Prometheus error rates, latency) against baseline.
- Automatically increments traffic (5% $\rightarrow$ 25% $\rightarrow$ 50% $\rightarrow$ 100%) or triggers an automated rollback.

### Essential Zero-Downtime Checklist
- **Graceful Shutdown (`SIGTERM`)**: App must capture `SIGTERM`, stop accepting new requests, finish in-flight requests, and exit cleanly before `terminationGracePeriodSeconds`.
- **PreStop Hook**: Add `sleep 5` to allow kube-proxy and Ingress controllers to remove the pod IP from iptables/Endpoints before the application process stops listening.
- **PodDisruptionBudgets (PDB)**: Enforces minimum pod availability (`minAvailable: 80%`) during node upgrades or auto-scaling events.
- **Backward-Compatible Database Migrations**: Use the **Expand and Contract (Parallel Run)** pattern. DB migrations must never drop or rename a column in the same release as the application update.

---

## 20. Production Outage: Step-by-Step Incident Debugging Framework

### The Senior SRE Incident Response Framework
When a production outage strikes, follow a disciplined protocol: **Triage $\rightarrow$ Mitigate $\rightarrow$ Investigate $\rightarrow$ Restore $\rightarrow$ Post-Mortem**.

```
[Phase 1: Triage & Incident Command (First 5 Minutes)]
├── Acknowledge incident & assign Incident Commander (IC)
├── Establish single communication war room (Slack channel & conference bridge)
└── Update public/internal status page to manage stakeholder communications
         │
         ▼
[Phase 2: Mitigate First, Find Root Cause Later!]
├── Ask: "WHAT CHANGED IN THE LAST 60 MINUTES?" (80% of outages stem from recent changes)
├── Check: Recent CI/CD deployments, feature flags, DNS edits, cloud provider notices
└── Quick-Action Mitigations:
    ├── Roll back recent deployment (helm rollback / git revert)
    ├── Disable suspected feature flag
    └── Scale out replicas or flush congested caches
         │
         ▼
[Phase 3: Systematic Outside-In Isolation]
├── Layer 1: DNS & CDN (Resolution failure? Expired TLS cert? CDN routing errors?)
├── Layer 2: Load Balancer (Healthy target counts? ALB 502/504 errors?)
├── Layer 3: Ingress / Mesh (Ingress controller saturation, rate limit exhaustion)
├── Layer 4: Compute & Cluster (kubectl get nodes, OOMKilled pods, Evictions)
├── Layer 5: Application Logs & Traces (Stack traces in OpenSearch, error spikes in APM)
└── Layer 6: Persistence & Backends (DB connection pools, locks, Redis memory, 3rd party APIs)
         │
         ▼
[Phase 4: Verification & Resolution]
├── Confirm Golden Signals (Latency, Traffic, Errors, Saturation) return to normal
└── Close incident and update status page
         │
         ▼
[Phase 5: Blameless Post-Mortem]
├── Timeline of events from first alert to full recovery
├── 5 Whys Root Cause Analysis
└── Action items assigned to owners with Jira tracking (prevent recurrence)
```

---
*Guide compiled for Senior & Lead DevOps / SRE Technical Interviews.*
