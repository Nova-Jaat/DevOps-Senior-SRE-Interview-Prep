# PTC Senior DevOps Engineer Interview Preparation Guide

A comprehensive, production-grade master reference for the PTC Senior DevOps Engineer technical rounds, structured with real-world architectural context, diagnostic commands, and senior-level troubleshooting workflows.

---

## 📌 Round 1: Core Architecture & Foundational Engineering

### 1. Project Architecture and CI/CD Pipeline
#### Architecture Overview (Production Microservices)
* **Compute:** AWS Elastic Kubernetes Service (EKS) spanning multiple Availability Zones (AZs) managed with Karpenter for JIT node provisioning.
* **Networking:** Multi-tier VPC with public subnets (ALB, NAT Gateways) and private subnets (EKS worker nodes, databases). AWS ALB Ingress Controller routing traffic to internal services.
* **Data Layer:** Amazon RDS PostgreSQL (Multi-AZ) for transactional data, Amazon ElastiCache Redis for caching and session management, and S3 for object storage.
* **Security & Observability:** AWS IAM Roles for Service Accounts (IRSA), HashiCorp Vault / AWS Secrets Manager, Prometheus + Grafana (Mimir/Thanos for long-term storage), and AWS OpenSearch/Fluentbit for logging.

#### CI/CD Pipeline Workflow (GitOps Model)
```
[Developer Push] 
       │
       ▼
[GitLab CI / GitHub Actions] ──> Lint, Unit Test, SAST (SonarQube)
       │
       ▼
[Container Build] ─────────────> Multi-stage Docker build, Trivy scan
       │
       ▼
[Push to ECR] ─────────────────> Tag with Git SHA & semver, Image signed with Cosign
       │
       ▼
[Config Repo Update] ──────────> Update Helm values / Kustomize image tag via PR/Bot
       │
       ▼
[ArgoCD / GitOps Controller] ──> Pulls changes, detects drift, synchronizes to EKS Cluster
       │
       ▼
[Progressive Rollout] ─────────> Canary deployment via Argo Rollouts / Flagger with metric analysis
```

---

### 2. Docker & Container Internals
* **What is Docker?** Docker is a containerization platform that leverages Linux kernel features to package an application and all its dependencies into an isolated, portable unit.
* **Kernel Primitives:**
  * **Namespaces:** Provide isolation (PID for processes, NET for network interfaces/routing, MNT for filesystem mount points, IPC for inter-process communication, UTS for hostname, USER for user/group IDs).
  * **Cgroups (Control Groups):** Enforce resource constraints and metering (CPU, memory, disk I/O, network bandwidth).
  * **UnionFS / Overlay2:** Layered copy-on-write storage driver allowing shared base layers and lightweight ephemeral read-write layers.
* **Image vs Container:**
  * **Image:** An immutable, read-only template built from ordered filesystem layers (defined by a Dockerfile) representing a snapshot of binaries, libraries, and metadata.
  * **Container:** An active runtime instance of an image. It adds a thin, writable top layer (container layer) on top of the immutable image layers and runs as an isolated process on the host kernel.

---

### 3. Kubernetes Architecture
Kubernetes follows a master-worker (Control Plane - Data Plane) distributed architecture:

#### Control Plane Components
1. **`kube-apiserver`:** The cluster's central entry point. Exposes the REST API, handles authentication/authorization (RBAC), admission controllers, and persists state to etcd.
2. **`etcd`:** A consistent, highly-available distributed key-value store used as Kubernetes' single source of truth for all cluster state, spec, and metadata.
3. **`kube-scheduler`:** Watches for newly created unscheduled pods and assigns them to optimal nodes based on resource requests/limits, affinities/anti-affinities, taints/tolerations, and topology spread constraints.
4. **`kube-controller-manager`:** Runs core reconciliation control loops (Node Lifecycle Controller, ReplicaSet Controller, EndpointSlice Controller, ServiceAccount Controller).
5. **`cloud-controller-manager`:** Integrates with underlying cloud provider APIs (managing cloud load balancers, node routing, EBS volume provisioning).

#### Worker Node (Data Plane) Components
1. **`kubelet`:** The primary node agent. Watches PodSpecs via the API server, instructs the container runtime via CRI (Container Runtime Interface) to start/stop containers, and reports node and pod health.
2. **`kube-proxy`:** Manages network routing rules on nodes (using `iptables` or `IPVS`) to implement the Kubernetes Service abstraction (ClusterIP, NodePort) and enable pod-to-service load balancing.
3. **Container Runtime:** Container engine complying with CRI (e.g., `containerd`, `CRI-O`) responsible for pulling images, running containers, and managing storage/network interfaces via CNI.

---

### 4. Pod, Service, and Deployment
* **Pod:** The smallest deployable computing unit in Kubernetes. Represents a single instance of an application workload running one or more tightly coupled containers sharing the same network namespace (IP address, port space) and storage volumes (IPC, IPC shared memory).
* **Service:** An abstract, stable network abstraction representing a set of ephemeral pods. Decouples the frontend from backend pod lifecycle changes using label selectors.
  * *Types:* `ClusterIP` (internal default), `NodePort` (static port on every node), `LoadBalancer` (provisions cloud provider LB), `ExternalName` (DNS CNAME).
* **Deployment:** A higher-level declarative controller that manages `ReplicaSets`. Provides automated rolling updates, declarative rollbacks, zero-downtime version transitions, self-healing, and horizontal scaling.

---

### 5. Deployment Strategies: Rolling, Blue-Green, Canary

| Strategy | Mechanism | Pros | Cons / Trade-offs |
| :--- | :--- | :--- | :--- |
| **Rolling** | Progressively replaces old pods with new ones using `maxSurge` and `maxUnavailable`. | Zero downtime; no extra infrastructure needed. | Both versions run concurrently in production; backward compatibility required in DB. |
| **Blue-Green** | Maintains two identical environments: Blue (active/current) and Green (new release). Router/Service points to Green once validated. | Instant cutover and near-instantaneous rollback (flip router back). | High cost (requires 2x resource capacity during release); DB migrations must support both. |
| **Canary** | Routes a tiny fraction of live user traffic (e.g., 5%, 10%) to the new version before rolling it out to 100%. | Real-world validation with minimal blast radius; automated rollback on metric thresholds (P99, error rate). | Requires advanced ingress/mesh routing (e.g., Argo Rollouts, Istio, Linkerd) and metric monitoring. |

---

### 6. Terraform & State File
* **What is Terraform?** An open-source, declarative Infrastructure as Code (IaC) tool that manages cloud and on-premise resources using HashiCorp Configuration Language (HCL).
* **What is the State File (`terraform.tfstate`)?**
  * Serves as the **source of truth** mapping declarative code definitions to real-world cloud resource IDs and attributes.
  * Tracks metadata and resource dependencies for building directed acyclic graphs (DAGs) to determine creation/destruction order.
  * Caches resource attributes to speed up `plan` operations and compute drift.
* **Production Best Practices:**
  * **Remote Backend:** Store state in Amazon S3 or Google Cloud Storage with versioning and encryption at rest enabled.
  * **State Locking:** Use DynamoDB (or S3 native locking in TF 1.10+) to prevent concurrent executions from corrupting state.
  * **Separation of Concerns:** Break state into small, modular workspaces/stacks (e.g., networking, compute, database) to limit blast radius.

---

### 7. Public vs. Private Subnets in AWS
* **Public Subnet:**
  * Associated route table has an explicit route pointing to an **Internet Gateway (IGW)**: `0.0.0.0/0 -> igw-xxxx`.
  * Hosts resources needing direct internet reachability (Application Load Balancers, Bastion Hosts / NAT Gateways). Resources can acquire public IPs.
* **Private Subnet:**
  * Route table has **no direct route to IGW**. Outbound internet access is routed via a **NAT Gateway** located in a public subnet: `0.0.0.0/0 -> nat-xxxx`, or uses VPC Endpoints (PrivateLink) for AWS services.
  * Hosts backend workloads (EKS worker nodes, databases, microservices) with no public IP exposure.

---

### 8. AWS IAM: Role vs. Policy
* **IAM Policy:** A JSON document explicitly defining permissions. Uses the format:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
    }]
  }
  ```
  Policies do not have identity; they must be attached to an identity (user, group, or role) or resource.
* **IAM Role:** An identity created in AWS that has specific permissions but is not tied to a single user or permanent password.
  * Relies on **Trust Policies** (`sts:AssumeRole`) defining *who* can assume the role.
  * When assumed, the AWS Security Token Service (STS) returns **temporary, short-lived security credentials** (access key, secret key, session token) with automated expiry.
  * Used for EC2 Instance Profiles, Lambda execution roles, and EKS IRSA (IAM Roles for Service Accounts).

---

### 9. Load Balancers & How They Work
* **Core Concept:** Distributes incoming network/application traffic across a pool of backend targets (EC2 instances, containers, IP addresses) across multiple AZs to prevent overload, maximize throughput, and ensure fault tolerance.
* **Operational Flow:**
  1. Client sends request to the load balancer's DNS record.
  2. Load Balancer receives request, terminates/inspects TLS, and evaluates listener rules.
  3. Periodically runs **health checks** against target groups (e.g., `GET /healthz`). Targets failing health thresholds are automatically deregistered.
  4. Distributes traffic using algorithms (Round Robin, Least Outstanding Requests, Flow Hash).
* **AWS ALB (L7) vs NLB (L4):**
  * **ALB (Application Load Balancer):** Layer 7 (HTTP/HTTPS/gRPC). Inspects headers, hostnames, paths, query params. Supports path-based routing, redirect rules, and sticky sessions.
  * **NLB (Network Load Balancer):** Layer 4 (TCP/UDP/TLS). Ultra-high throughput, handles millions of requests/sec with microsecond latency, preserves source IP, provides static elastic IPs per AZ.

---

### 10. Git Branching Strategy
* **Trunk-Based Development (Senior / Modern DevOps Choice):**
  * Developers merge small, frequent updates directly into a single main branch (`trunk` or `main`).
  * Features are developed in short-lived branches (< 1-2 days) protected by automated CI gates (unit tests, security scans).
  * Feature Flags (Toggles) are used to decouple deployment from release, enabling true Continuous Deployment.
* **GitFlow (Alternative for Scheduled Release Cycles):**
  * Branches: `main`, `develop`, `feature/*`, `release/*`, `hotfix/*`.
  * Overhead: Merge conflicts, long-lived branches, complex cherry-picking, slows down continuous delivery.

---

### 11. Monitoring Applications in Production (Observability Stack)
A senior strategy implements the **Three Pillars of Observability** plus **Alerting** adhering to Google SRE **Four Golden Signals** (Latency, Traffic, Errors, Saturation):
1. **Metrics:** Time-series telemetry via **Prometheus / VictoriaMetrics / Datadog**.
   * Infrastructure: Node CPU, memory, network I/O, disk pressure.
   * Workload: Pod restart counts, request rate, P95/P99 latency, HTTP 5xx rates.
   * Visualization: **Grafana** dashboards.
2. **Logs:** Centralized aggregation via **Fluentbit / Vector -> OpenSearch / Loki / CloudWatch**.
   * Structured JSON logs with correlation IDs (`trace_id`, `request_id`).
3. **Distributed Tracing:** **OpenTelemetry (OTel) + Jaeger / AWS X-Ray**.
   * Tracks request lifecycles across microservice boundaries to pinpoint network bottlenecks and database delays.
4. **Alerting & Incident Management:**
   * **Prometheus Alertmanager / PagerDuty** with SLO/SLI-based alerts (error budget burn rates) rather than noisy low-signal alerts.

---

### 12. DNS Architecture and Resolution Flow
* **What is DNS?** The Domain Name System translates human-friendly domain names (`api.company.com`) into computer-routable IP addresses (`192.0.2.1` or `2600::`).
* **Resolution Steps (Recursive Lookup):**
  1. **Client / OS Cache:** Browser checks internal cache -> OS resolver checks local `/etc/hosts` and local DNS cache.
  2. **Resolving Name Server (Recursive Resolver):** Query forwarded to ISP or public DNS (8.8.8.8, 1.1.1.1).
  3. **Root Nameserver (`.`):** Directs the resolver to the appropriate Top-Level Domain (TLD) server.
  4. **TLD Nameserver (`.com`):** Directs to the Authoritative Nameserver responsible for the domain.
  5. **Authoritative Nameserver (e.g., AWS Route 53):** Looks up the record set and returns the IP address.
  6. **Caching:** The recursive resolver caches the record according to its **TTL (Time to Live)** and hands the answer to the client.
* **Essential Record Types:** `A` (IPv4), `AAAA` (IPv6), `CNAME` (canonical name alias), `ALIAS` (AWS Route53 zone apex alias), `MX` (mail exchange), `TXT` (SPF/DKIM/verification).

---

## 📌 Round 2: Deep Troubleshooting & Senior Scenarios

### 1. Scaling an Application in Kubernetes
A resilient Kubernetes scaling architecture requires a layered approach across pods, nodes, and events:
1. **Horizontal Pod Autoscaler (HPA):**
   * Adjusts pod replicas based on CPU/Memory utilization or custom application metrics (e.g., HTTP request rate from Prometheus via Prometheus Adapter).
   * Relies on Kubernetes Metrics Server.
2. **Vertical Pod Autoscaler (VPA):**
   * Automatically analyzes historical resource usage and adjusts CPU/Memory `requests` and `limits` for workloads that cannot scale horizontally.
3. **Cluster Node Autoscaling:**
   * **Karpenter (Modern AWS EKS Standard):** Directly provisions right-sized EC2 instances within seconds based on pending pod requirements without node group constraints.
   * **Cluster Autoscaler (Legacy):** Scales EC2 Auto Scaling Groups when pods are unschedulable due to insufficient resources.
4. **Event-Driven Autoscaling (KEDA):**
   * Scales pods based on external queue depths (AWS SQS, Kafka, RabbitMQ) even down to zero.

---

### 2. Securing Container Images in Kubernetes
* **Base Image Hardening:** Use minimal, verified base images (e.g., Google Distroless, Alpine, Chainguard) to minimize CVE attack surfaces.
* **Non-Root Execution:** Enforce `USER nonroot` in Dockerfiles and specify `securityContext.runAsNonRoot: true`, `readOnlyRootFilesystem: true`, and `allowPrivilegeEscalation: false` in PodSpecs.
* **Multi-Stage Builds:** Strip build tools (compilers, git, SDKs) from the final runtime container image.
* **Automated CI Scanning:** Scan images at build time using Trivy, Grype, or Snyk with gates blocking builds on High/Critical vulnerabilities.
* **Supply Chain & Registry Security:**
  * Sign images using **Cosign (Sigstore)** and store attestations (SBOM via Syft).
  * Use private registries (ECR) with immutable tags and private link endpoints.
* **Admission Control:** Deploy **Kyverno** or **OPA Gatekeeper** to block unverified images, unsigned digests, or images running as root from being admitted to the cluster.

---

### 3. Role of Amazon CloudFront in an Architecture
* **Global Edge Acceleration:** Caches static assets (images, JS, CSS, videos) across 600+ Points of Presence (PoPs), terminating TCP/TLS handshakes close to end-users to reduce P99 latency.
* **Origin Offloading:** Absorbs high-volume traffic bursts, protecting origin services (ALBs, S3, EC2) from saturation and reducing egress bandwidth costs.
* **DDoS & Perimeter Security:** Deeply integrated with **AWS Shield** (DDoS mitigation) and **AWS WAF** (rate limiting, SQLi/XSS blocking, geo-blocking, bot control).
* **Edge Compute (Lambda@Edge / CloudFront Functions):** Executes lightweight code at the edge for URL rewrites, A/B testing, auth token validation, and custom HTTP security headers (HSTS, CSP).
* **High Availability & Origin Failover:** Configured with origin groups to automatically fail over to a backup bucket/endpoint if primary origin returns 5xx errors.

---

### 4. Designing Disaster Recovery (DR) for Amazon RDS
DevOps leaders define DR based on **RPO (Recovery Point Objective)** and **RTO (Recovery Time Objective)**:

```
[Primary Region: us-east-1]                      [Secondary Region: us-west-2]
┌───────────────────────────────┐               ┌───────────────────────────────┐
│ Amazon RDS Multi-AZ (Active)  │               │ Cross-Region Read Replica     │
│ ┌──────────┐   Synchronous   │               │ (Asynchronous replication)    │
│ │ Primary  │ ───────────────> │               │             │                 │
│ └──────────┘                 │               │             ▼ (On Disaster)   │
└───────────────────────────────┘               │ ┌───────────────────────────┐ │
                 │                              │ │ Promoted to Standalone DB │ │
                 │ Automated Snapshot Copy      │ └───────────────────────────┘ │
                 └─────────────────────────────>└───────────────────────────────┘
```

1. **High Availability (HA within Region):**
   * Multi-AZ deployment: Synchronous physical replication to a standby instance in an alternate AZ. Automatic failover in 60-120 seconds if the primary fails (RPO = 0, RTO = 1-2 mins).
2. **Cross-Region Disaster Recovery:**
   * **Cross-Region Read Replica:** Asynchronous replication to a secondary AWS region. During a regional outage, promote the read replica to a standalone writable instance via automated script/Terraform (RPO: low seconds, RTO: 5-10 minutes).
   * **Aurora Global Database (If using Aurora):** Uses storage-level cross-region replication (<1 sec lag); failover takes under 1 minute with no data loss.
3. **Automated Cross-Region Snapshot Backups:**
   * Automated snapshot policies via **AWS Backup**, copying snapshots to a secondary account/region encrypted with a cross-region KMS CMK.
   * Enable Point-in-Time Recovery (PITR) with transaction log archiving.

---

### 5. Troubleshooting: Developer Cannot Access an S3 Bucket
Follow the S3 Authorization Evaluation Logic systematically:

1. **IAM Identity-Based Policies:**
   * Verify the developer's IAM user/role has explicit `s3:GetObject`, `s3:PutObject`, or `s3:ListBucket` permissions.
   * Check for any explicit `"Effect": "Deny"` in other attached policies or permission boundaries.
2. **S3 Bucket Policy:**
   * Inspect the bucket policy for IP restrictions, VPC restrictions, or principal restrictions that might block the user.
3. **S3 Block Public Access (BPA):**
   * Verify if BPA settings at the account or bucket level conflict with the required access.
4. **AWS KMS Key Policy:**
   * If the bucket is encrypted with SSE-KMS (Customer Managed Key), the user **must** have `kms:Decrypt` and `kms:GenerateDataKey` permissions on the KMS key policy, not just S3 access.
5. **AWS Organizations SCP (Service Control Policy):**
   * Check if a parent OU or root SCP denies S3 actions or restricts operations to specific regions.
6. **VPC Endpoint Policy (If accessing from inside VPC):**
   * If traffic traverses an S3 Gateway Endpoint, verify the endpoint policy allows the identity and action.
7. **Object Ownership:**
   * Check if objects were written by another AWS account without granting bucket owner full control (resolve via S3 Object Ownership: `BucketOwnerEnforced`).

---

### 6. Troubleshooting: Latency Issues in an EKS Cluster
Use a top-down, layer-by-layer methodology:

```
[Ingress / ALB] ──> [Node & OS] ──> [CoreDNS / Network] ──> [Pod / Application] ──> [Dependencies]
```

1. **Ingress & Load Balancer Layer:**
   * Check AWS CloudWatch metrics for ALB: `TargetResponseTime` vs `RequestCount`. If `TargetResponseTime` is high, the delay is inside the cluster, not the ALB.
2. **Node-Level Saturation:**
   * Run `kubectl top nodes` / check node exporter metrics.
   * Look for CPU/Memory saturation, disk I/O bottlenecks, and network ENI bandwidth throttling.
   * Check for **Linux conntrack table exhaustion** (`net.netfilter.nf_conntrack_count`) which drops packets silently.
3. **Kubernetes Networking & CoreDNS:**
   * CoreDNS throttling: High pod DNS query rates can exhaust CoreDNS pods. Check CoreDNS response latency and deploy `NodeLocal DNSCache`.
   * AWS VPC CNI: Inspect IP address availability and ENI allocation delays.
4. **Pod Level & Cgroups Throttling:**
   * Check pod CPU throttling: `container_cpu_cfs_throttled_periods_total`. Even if CPU usage is below 100%, aggressive CPU `limits` cause kernel CFS quota throttling.
5. **Downstream & Database Bottlenecks:**
   * Trace calls with OpenTelemetry / Jaeger. Verify if latency originates from slow database queries, connection pool starvation, or external API timeouts.

---

### 7. Troubleshooting: AWS Lambda Failing Internally
1. **CloudWatch Metrics Analysis:**
   * Check `Errors`, `Throttles`, and `Duration` graphs.
   * High `Duration` near timeout limit indicates deadlocks, unhandled network timeouts, or heavy cold starts.
2. **CloudWatch Logs Insights Queries:**
   * Filter logs for `REPORT` lines:
     ```sql
     filter @type = "REPORT"
     | fields @timestamp, @duration, @billedDuration, @memorySize, @maxMemoryUsed
     | sort @timestamp desc
     ```
   * If `@maxMemoryUsed` equals `@memorySize`, the Lambda ran out of memory (OOM causes instant process termination without runtime stack trace).
   * Check for `Task timed out after X.XX seconds` or `Process exited before completing request`.
3. **VPC Configuration & Networking Issues:**
   * If Lambda is in a VPC private subnet, it cannot reach the public internet unless that subnet routes through a **NAT Gateway**.
   * Check Security Group outbound rules (must allow egress to the target service/port).
4. **Code Initialization Errors:**
   * Errors occurring outside the handler function (database connection initialization, global variable loading) fail before execution starts.
5. **IAM Execution Role:**
   * Ensure the role has permissions to publish logs (`AWSLambdaBasicExecutionRole`) and access downstream resources.

---

### 8. Troubleshooting: CI/CD Pipeline Unable to Connect to GitLab
1. **Network Connectivity & Routing:**
   * From the CI runner host, test connectivity:
     ```bash
     curl -v https://gitlab.company.com/health
     nc -zv gitlab.company.com 443
     ```
   * Check DNS resolution, corporate proxy settings (`HTTP_PROXY`, `NO_PROXY`), and security group / firewall rules blocking outbound 443.
2. **Runner Registration & Authentication:**
   * Check runner daemon status: `sudo gitlab-runner status`.
   * Verify runner authentication token: `sudo gitlab-runner verify`. If the registration token was reset or expired, the runner shows disconnected.
3. **TLS/SSL Certificate Verification:**
   * If GitLab uses a private CA or self-signed cert, check if the runner's trust store includes the root certificate (`/etc/gitlab-runner/certs/gitlab.crt`).
   * Check for `x509: certificate signed by unknown authority`.
4. **Git Credentials & SSH Keys:**
   * If using Git over SSH in pipelines, verify deploy keys or CI job tokens (`CI_JOB_TOKEN`) have read/write access to the repository.
5. **Rate Limiting & Server Load:**
   * Check GitLab server Nginx logs (`/var/log/gitlab/nginx/gitlab_access.log`) for HTTP 429 (Rate Limited) or 502/503 responses.

---

### 9. Troubleshooting: Kubernetes Not Able to Expose a Pod
Step-by-step diagnostic checklist:

1. **Verify Pod Status & Readiness:**
   * Run `kubectl get pods -l app=my-app -o wide`.
   * A pod must be in `Running` state and have all containers passing their **Readiness Probe**. A pod that is running but not `Ready` will not be added to Service endpoints.
2. **Check Endpoints / EndpointSlices:**
   ```bash
   kubectl get endpoints <service-name>
   kubectl describe endpointslices <service-name>
   ```
   * If endpoints list is `<none>`, the service is not routing traffic to any pods.
3. **Verify Label Selector Matching:**
   * Inspect the service selector: `kubectl get svc <service-name> -o yaml | grep -A 5 selector`.
   * Match against pod labels: `kubectl get pods --show-labels`. Every label in `spec.selector` must be present on the pod.
4. **Verify Port Mappings:**
   * Ensure `service.spec.ports[].targetPort` matches the exact port the application process inside the container is listening on (not just `containerPort` in the spec).
5. **Inspect NetworkPolicies:**
   * Check if an active `NetworkPolicy` isolates the pod and blocks ingress traffic on that port from the ingress controller or namespace.
6. **Cloud Provider LoadBalancer Integration:**
   * If `type: LoadBalancer`, check `kubectl describe svc <service-name>` events for cloud controller errors (e.g., missing subnet tags, insufficient AWS IAM permissions).

---

### 10. Troubleshooting: Pods Running but Users Getting 503 Service Unavailable
HTTP 503 means the gateway/proxy cannot reach an upstream server capable of handling the request.

```
Client ──> [Cloud Load Balancer] ──> [Ingress Controller] ──> [K8s Service / Endpoints] ──> [App Pod]
                 ▲                            ▲                             ▲
            ALB Health                  Ingress Upstream               Pod Readiness / 
               Check                         Error                      Backpressure
```

1. **Check Service Endpoints:**
   * Run `kubectl get endpoints <svc>`. If endpoints are empty, readiness probes failed or pods were evicted.
2. **Ingress / ALB Target Group Health:**
   * Check AWS Target Group status in EC2 console or via AWS CLI:
     ```bash
     aws elbv2 describe-target-health --target-group-arn <arn>
     ```
   * If target health check path (`/healthz`) fails, the ALB marks targets `Unhealthy` and returns 503.
3. **Ingress Controller Logs (Nginx / Traefik / Envoy):**
   * Inspect controller logs: `kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx`.
   * Look for `upstream connect error`, `connection refused`, or `upstream timed out`.
4. **Application Concurrency & Connection Pools:**
   * The application process may be running, but its worker thread pool (e.g., Node.js event loop lag, Tomcat/Gunicorn worker starvation) is exhausted and rejecting new incoming TCP connections.
5. **Circuit Breaking / Service Mesh (If using Istio):**
   * Check if Envoy sidecar circuit breaker tripped due to consecutive 5xx errors or connection pool overflow.

---

### 11. Recovery: Terraform Apply Failed Midway
When Terraform fails halfway through an apply, **state drift and partial provisioning occur**.

#### Safe Recovery Protocol:
1. **Never Re-run Blindly:** Do not execute `terraform destroy` or re-run `apply` without analyzing the failure.
2. **Review the Error Message:** Identify which specific resource failed (e.g., API timeout, dependency failure, IAM quota exceeded).
3. **Handle State Locking:**
   * If the process crashed and left a lock in DynamoDB:
     ```bash
     terraform force-unlock <LOCK-ID>
     ```
   * Only run force-unlock after confirming no CI runner or teammate is actively executing against the state.
4. **Reconcile State with Cloud Reality:**
   * Run a refresh-only plan to sync the state file with actual cloud resources without making changes:
     ```bash
     terraform plan -refresh-only
     terraform apply -refresh-only
     ```
5. **Inspect Partially Created Resources:**
   * If a resource exists in the cloud but Terraform failed before recording it:
     ```bash
     terraform import <resource_type>.<resource_name> <cloud_id>
     ```
   * If a resource is corrupted or half-provisioned:
     ```bash
     terraform taint <resource_type>.<resource_name>
     ```
6. **Fix Root Cause & Re-plan:**
   * Resolve permission, parameter, or dependency issues in the code.
   * Run `terraform plan` to verify the execution graph cleanly recovers, then run `terraform apply`.

---

### 12. Secure Secret Management Across Multiple Environments
A zero-trust, enterprise secret management strategy:

```
┌───────────────────────────────────────┐
│ HashiCorp Vault / AWS Secrets Manager │
└───────────────────┬───────────────────┘
                    │ Automated Sync / Decryption
                    ▼
┌───────────────────────────────────────┐
│ External Secrets Operator (ESO)       │
└───────────────────┬───────────────────┘
                    │ Injects native K8s Secret
                    ▼
┌───────────────────────────────────────┐
│ Kubernetes Secret (tmpfs in memory)   │
└───────────────────┬───────────────────┘
                    │ Mounted as volume / env
                    ▼
┌───────────────────────────────────────┐
│ Application Pod (Non-root user)       │
└───────────────────────────────────────┘
```

1. **Central Secret Store:**
   * Store secrets externally in **AWS Secrets Manager** or **HashiCorp Vault**.
   * Organize using strict environment hierarchies (`/prod/payments/db_password`, `/dev/payments/db_password`).
2. **Kubernetes Integration (No Hardcoded Secrets):**
   * **External Secrets Operator (ESO):** Syncs secrets from AWS Secrets Manager / Vault into native Kubernetes secrets using IAM Roles for Service Accounts (IRSA).
   * **Secrets Store CSI Driver:** Mounts secrets directly from AWS Secrets Manager/Vault as a local volume into the container filesystem (`tmpfs` in-memory), preventing secrets from being written to disk or etcd.
3. **GitOps Secret Encryption (If kept in Git):**
   * Use **Mozilla SOPS** (encrypted with AWS KMS / age keys) or **Bitnami Sealed Secrets**. Only the encrypted ciphertext is committed to version control.
4. **Automated Rotation & Least Privilege:**
   * Configure automated rotation via AWS Secrets Manager Lambda rotators.
   * Restrict access via IAM policies scoped to individual service accounts.
   * Audit all access using AWS CloudTrail / Vault audit logs.

