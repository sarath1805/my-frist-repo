# Screening Interview — Questions & Answers
## Top 5 Skills: ArgoCD, AWS, EKS, Python, Ansible

This document contains all interview questions and answers for the screening call, covering the senior technical person's top skills.

---

## ArgoCD

### Q1. What is GitOps and how does ArgoCD implement it?

GitOps is an operating model where Git is the single source of truth for both infrastructure and application state. Every change goes through a Git commit — no direct `kubectl apply` in production. ArgoCD implements this with a controller that continuously watches a Git repo and compares the declared state to the live cluster state. If they differ, ArgoCD either alerts or automatically reconciles. This gives auditability through Git history, easy rollback by reverting commits, and consistency across clusters.

### Q2. Core ArgoCD components.

- **API Server**: exposes the gRPC/REST API for the UI, CLI, and webhooks
- **Repository Server**: clones Git repos, renders manifests (Helm template, Kustomize build), caches them
- **Application Controller**: the reconciliation engine — watches `Application` CRs, compares Git vs cluster, performs sync
- **Dex** (optional): handles SSO/OIDC integration
- **Redis**: caching layer for manifest rendering and state

The Application Controller is the heart — everything else feeds into it.

### Q3. Sync, Refresh, Auto-Sync, Self-Heal, Prune.

- **Refresh**: re-reads Git and cluster state to update ArgoCD's comparison view. No changes applied.
- **Sync**: applies the desired state from Git to the cluster.
- **Auto-Sync**: triggers Sync automatically whenever drift is detected.
- **Self-Heal**: with auto-sync enabled, reverts any manual changes in the cluster back to Git's state.
- **Prune**: deletes resources from the cluster that no longer exist in Git.

In production I usually enable auto-sync with self-heal and prune for non-critical apps, but require manual sync for stateful or sensitive workloads.

### Q4. Desired vs live vs target state?

- **Desired state**: what's declared in Git
- **Live state**: what currently exists in the cluster
- **Target state**: the rendered desired state after Helm template or Kustomize build is applied

ArgoCD compares target state with live state to determine sync status.

### Q5. Onboarding a new application.

1. Create a Helm chart or Kustomize directory in the team's Git repo
2. Add an `Application` CR pointing to the repo path, target cluster, namespace, and sync policy
3. Apply the CR via `kubectl apply` or through an App-of-Apps repo
4. ArgoCD picks it up, renders manifests, and either presents a diff for manual sync or auto-syncs

Best practice: never `kubectl apply` the application's resources directly — always go through ArgoCD.

### Q6. App of Apps pattern — when to use?

A parent ArgoCD `Application` that manages a set of child `Application` resources stored in Git. Use it for cluster bootstrapping — one root app deploys all platform components (Prometheus, cert-manager, external-secrets, team apps). It enables hierarchical, declarative management of an entire cluster's application landscape with a single entry point.

### Q7. ApplicationSet vs Application.

`Application` is a single declaration for one workload. `ApplicationSet` is a generator that produces multiple `Application` resources dynamically from a template. Use cases: deploying the same app across N clusters using the `clusters` generator, or one app per Git directory using the `git` generator. ApplicationSet is the right tool when you have repetitive deployments — without it you'd manually maintain dozens of Application YAMLs.

### Q8. Repo structure for multi-team, multi-environment.

I prefer:
- One **platform repo** containing the App-of-Apps root and platform-level Application manifests
- Per-team **app repos** containing their Helm charts and Kustomize overlays
- Per-environment overlays inside each app repo: `overlays/dev`, `overlays/staging`, `overlays/prod`

Separation of concerns: platform team owns infrastructure, app teams own their services. Environment promotion is a Git PR from `dev` overlay → `staging` → `prod`.

### Q9. Environment-specific Helm values.

I keep a base `values.yaml` and per-environment files: `values-dev.yaml`, `values-prod.yaml`. In the ArgoCD Application spec:

```yaml
source:
  helm:
    valueFiles:
      - values.yaml
      - values-prod.yaml
```

For secret-like values, I use external-secrets, not value files.

### Q10. How does ArgoCD handle Helm, Kustomize, plain manifests?

ArgoCD auto-detects the source type. For Helm, it runs `helm template` to render manifests — it doesn't use Tiller or `helm install`, which means Helm hooks behave differently (ArgoCD translates them to its own sync hooks). For Kustomize, it runs `kustomize build`. For plain manifests, it just applies them directly. All three are normalized into the same diff/sync flow internally.

### Q11. Sync waves and hooks — real use case.

Sync waves are annotations (`argocd.argoproj.io/sync-wave: "1"`) that order resource creation. Hooks (`PreSync`, `Sync`, `PostSync`, `SyncFail`) define resources that run at specific phases. Real example: a database schema migration `Job` annotated as `PreSync` runs before the new application Deployment. This ensures the schema is updated before the new code starts, and if migration fails, the sync aborts.

### Q12. Secret management in GitOps.

Three patterns:
1. **External Secrets Operator** (my preference): an `ExternalSecret` CR in Git references a secret stored in AWS Secrets Manager. ESO fetches and creates the Kubernetes Secret at runtime.
2. **Sealed Secrets**: encrypts secrets with a cluster-specific key. The encrypted `SealedSecret` is safe in Git.
3. **ArgoCD Vault Plugin**: substitutes placeholders from HashiCorp Vault during manifest rendering.

ESO decouples secret rotation from Git commits and integrates cleanly with AWS Secrets Manager.

### Q13. RBAC in ArgoCD.

ArgoCD has its own RBAC system layered on top of authentication. You define roles in the `argocd-rbac-cm` ConfigMap:

```yaml
policy.csv: |
  p, role:team-payments, applications, sync, payments/*, allow
  g, payments-team-group, role:team-payments
```

This grants the `payments-team-group` SSO group sync rights only on apps in the `payments` AppProject. AppProjects are the unit of multi-tenancy in ArgoCD — they restrict which clusters, namespaces, and source repos a team can use.

### Q14. SSO integration.

ArgoCD supports OIDC directly or via Dex. I configure it with an OIDC provider (Azure AD, Okta, AWS IAM Identity Center). Users authenticate via SSO and group claims from the OIDC token map to ArgoCD RBAC roles. No more shared `admin` accounts — every action is attributed to a user.

### Q15. OutOfSync but Healthy.

The application is running fine and serving traffic, but the live cluster state doesn't match Git. Common causes: someone manually edited a resource, a mutating admission webhook added fields, or auto-sync is disabled and a new Git commit hasn't been applied yet. Healthy means the workload is functioning regardless of sync state.

### Q16. Synced but Degraded.

Git was applied successfully, but the resulting workload is unhealthy — pods crashing, deployment failed progress check, or a custom health check failing. This is the most common production scenario: the deployment went through but the new pods can't start. ArgoCD did its job; the application has a problem.

### Q17. Debugging a stuck Progressing app.

1. `argocd app get <app>` — shows which resources are not synced or unhealthy
2. `kubectl get events -n <namespace> --sort-by=.lastTimestamp` — recent events explain most stuck states
3. Common causes: a PreSync job failed silently, a Deployment is rolling but new pods crash, a webhook is rejecting resources, PVC not binding
4. Check `kubectl describe pod` on any failed pods
5. If a hook job is stuck: `kubectl delete job <hook>` to let ArgoCD retry
6. Force refresh: `argocd app get <app> --hard-refresh`

### Q18. Rolling back a failed deployment.

Two options:
1. **Git revert** (the GitOps way): `git revert <commit> && git push` — ArgoCD picks up the revert and rolls back. Audit trail preserved.
2. **ArgoCD CLI**: `argocd app rollback <app> <revision>` — rolls back to a previous deployment revision in ArgoCD's history without changing Git.

I prefer Git revert because it keeps Git as the source of truth. The CLI rollback is useful for emergencies but creates drift between Git and live state.

### Q19. Drift detection and manual changes.

ArgoCD continuously detects drift between Git and the cluster. With `selfHeal: true`, it reverts manual changes automatically. Without it, the app shows OutOfSync and waits for explicit sync. My policy: production has self-heal enabled because manual changes are a process violation. If someone needs to make an emergency change, they make it in Git first, then ArgoCD applies it.

### Q20. Dev → staging → prod promotion.

Pattern I use:
1. Each environment is a separate ArgoCD Application pointing to its own overlay/values file
2. CI/CD builds the image and writes the new tag into the `dev` values file (auto-promotion)
3. `dev` → `staging` is a PR that updates the image tag in the staging values
4. `staging` → `prod` is another PR, gated by approval
5. Tools like Argo CD Image Updater or Argo Rollouts can automate parts of this

The image tag is the only thing flowing forward — config differences stay environment-specific.

### Scenario Q21. App constantly OutOfSync but Git isn't changing.

Something in the cluster is modifying resources after ArgoCD applies them. Suspects:
- Mutating admission webhooks (Istio sidecar injector, OPA Gatekeeper) adding fields
- HPA modifying `replicas` on a Deployment (fix: exclude `replicas` from diff using `ignoreDifferences`)
- Operators reconciling resources differently than Git defines
- Cluster autoscaler adding annotations

Solution: use `ignoreDifferences` in the Application spec to ignore fields managed by other controllers.

### Scenario Q22. Synced but pods crashing, traffic failing.

1. `kubectl logs <pod> --previous` — what's the crash cause?
2. Check exit code: 137 (OOM), 1 (app error), 139 (segfault)
3. Compare ConfigMaps/Secrets between working and broken state
4. Roll back via `git revert` immediately — fix root cause separately
5. Bring in the app owner — ArgoCD did its job; this is an application issue

### Scenario Q23. PreSync migration job failing.

1. `kubectl logs job/<hook-job>` — see why migration failed
2. Common causes: DB connection issue (network policy, credentials), schema conflict, timeout
3. Fix the migration script or DB connectivity
4. `kubectl delete job <hook-job>` — ArgoCD retries on next sync
5. If migration is broken: revert the offending commit in Git so ArgoCD stops attempting to apply the bad migration

### Scenario Q24. Same app across 40 clusters with minor differences.

Use `ApplicationSet` with the cluster generator:

```yaml
generators:
- clusters:
    selector:
      matchLabels:
        env: production
template:
  metadata:
    name: '{{name}}-myapp'
  spec:
    source:
      helm:
        valueFiles:
        - values-{{metadata.labels.region}}.yaml
```

Cluster-specific labels feed into the template. Adding a new cluster automatically deploys the app — no manual Application creation per cluster.

### Scenario Q25. Developer made an emergency manual change; self-heal reverted it.

Operationally I'd handle it in two parts:
1. **Immediate**: pause ArgoCD self-heal for that app (`argocd app set --self-heal=false`), let the dev re-apply the fix, validate
2. **Process**: emergency changes must still go through Git. The dev should commit the change to Git immediately so ArgoCD can sync it. If Git PR review is too slow for emergencies, set up a "break-glass" branch with a fast-track approval path. The principle is non-negotiable: Git is the source of truth, even for emergencies.

---

## AWS

### Q1. Security Groups vs NACLs.

- **Security Groups**: stateful, instance-level. Allow inbound implies return traffic is allowed. Default deny — only allow rules exist.
- **NACLs**: stateless, subnet-level. Need explicit allow for both inbound and outbound. Support deny rules processed in order.

SGs are my primary tool. NACLs are for coarse-grained subnet protection — like blocking a known bad IP range.

### Q2. IAM user, role, group, policy.

- **User**: a permanent identity with credentials (console password, access keys)
- **Group**: a collection of users — attach policies to groups instead of users individually
- **Role**: an identity that can be assumed via STS, providing temporary credentials. No permanent credentials attached.
- **Policy**: JSON document defining permissions — attached to users, groups, or roles

Best practice: humans use SSO mapped to roles; services use roles via instance profiles or IRSA. No long-lived access keys in production.

### Q3. IAM roles for EC2 and EKS workloads.

EC2: attach an instance profile (a wrapper around an IAM role). The EC2 metadata service provides temporary credentials to applications via `http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>`.

EKS: use IRSA — annotate a Kubernetes Service Account with the role ARN. Pods using that SA get temporary AWS credentials via OIDC token exchange. This is per-pod, not per-node — far better isolation than node-level instance profiles.

### Q4. VPC building blocks.

- **VPC**: isolated virtual network with a CIDR block
- **Subnets**: subdivisions of the CIDR, each in one AZ
- **Route tables**: define traffic routing per subnet
- **Internet Gateway**: enables internet access for public subnets
- **NAT Gateway**: enables outbound internet for private subnets
- **Security Groups & NACLs**: traffic filtering
- **VPC Endpoints**: private connectivity to AWS services without going through the internet

### Q5. Public vs private subnets.

A subnet is "public" if its route table has a route `0.0.0.0/0 → Internet Gateway`. Private subnets route outbound through a NAT Gateway and have no direct inbound from the internet. Public subnets host load balancers and bastion hosts; private subnets host application workloads and databases.

### Q6. IGW, NAT Gateway, route tables.

- **IGW**: bidirectional gateway between the VPC and the public internet. Required for public subnets.
- **NAT Gateway**: allows private subnet instances to initiate outbound connections (e.g., to pull container images) without exposing them to inbound internet traffic. AWS-managed, scales automatically.
- **Route tables**: per-subnet rules mapping destination CIDR to a target (IGW, NAT, VPC Endpoint, peering connection, Transit Gateway).

### Q7. ALB vs NLB vs CLB.

- **ALB**: Layer 7, HTTP/HTTPS, path/host routing, WebSocket support
- **NLB**: Layer 4, TCP/UDP/TLS, ultra-low latency, millions of connections, preserves client IP
- **CLB**: legacy, supports both L4 and L7 with limited features — avoid for new workloads

I use ALB for microservices and NLB for MQTT (TCP) or when extreme connection scale is needed.

### Q8. Route 53 routing policies.

- **Weighted**: split traffic by percentage — useful for canary deployments
- **Latency**: route users to the region with lowest latency from their location
- **Failover**: primary/secondary based on health checks
- **Geolocation**: route based on user's country/continent
- **Multivalue answer**: return multiple healthy IPs (basic DNS-level load balancing)

For active-passive DR, I use failover with health checks. For active-active multi-region, latency-based.

### Q9. S3 vs EBS vs EFS.

- **S3**: object storage, accessed via API, unlimited scale, eventually consistent strong-read-after-write
- **EBS**: block storage attached to one EC2 instance, like a virtual disk
- **EFS**: NFS-based shared file system, mountable by multiple instances/pods simultaneously

EBS for databases needing block I/O. EFS for shared file storage across pods. S3 for everything else — backups, artifacts, static assets, data lake.

### Q10. RDS vs self-managed databases.

RDS handles backups, patching, replication, failover automatically. Self-managed gives full control over configuration, extensions, and tuning, but you own all operational work — backups, OS patches, HA setup. I use RDS for most relational workloads. Self-managed only when RDS doesn't support what we need (specific Postgres extensions, ClickHouse, custom replication setups).

### Q11. Auto Scaling Groups.

ASGs maintain a desired count of EC2 instances and scale based on policies. Scaling can be driven by:
- CloudWatch metrics (CPU, memory, custom metrics)
- Target tracking policies (e.g., keep average CPU at 60%)
- Scheduled actions
- Predictive scaling based on historical patterns

ASGs handle instance replacement on failure and integrate with load balancer target groups.

### Q12. CloudWatch.

- **Metrics**: time-series numeric data — built-in for AWS services, custom metrics via API
- **Logs**: log aggregation with Log Groups and Log Streams
- **Dashboards**: visualizations combining metrics and log queries
- **Alarms**: threshold-based alerts triggering SNS, Lambda, or Auto Scaling actions
- **Logs Insights**: query language for logs

For Kubernetes I usually pair CloudWatch with Prometheus/Grafana — CloudWatch for AWS infrastructure, Prometheus for cluster and app metrics.

### Q13. Highly available app across AZs.

- VPC spans 3 AZs with subnets in each
- Application instances or pods spread across AZs (ASG `AvailabilityZones` set, or Kubernetes topology spread constraints)
- Multi-AZ RDS with synchronous standby in another AZ
- ALB or NLB across all AZs
- S3 (automatically multi-AZ within a region)
- Health checks remove unhealthy AZs from rotation

The principle: no single AZ failure should cause user-visible impact.

### Q14. Securing an AWS account for production.

- Root account locked away with MFA, never used operationally
- AWS Organizations with separate accounts per environment
- SSO via IAM Identity Center, no IAM users for humans
- Least privilege via IAM roles and Permission Boundaries
- CloudTrail enabled in all accounts, logs centralized
- GuardDuty and Config enabled
- VPC Flow Logs for network audit
- S3 buckets default to block-public-access
- Secrets in AWS Secrets Manager, never in code
- Service Control Policies (SCPs) at the OU level to enforce guardrails

### Q15. CloudTrail, Config, GuardDuty.

- **CloudTrail**: records API calls — who did what, when. Foundation for security audit.
- **Config**: tracks resource configuration changes over time and evaluates compliance against rules
- **GuardDuty**: threat detection using ML on CloudTrail, VPC Flow Logs, DNS logs — flags suspicious activity like crypto-mining, credential exfiltration

Together they provide audit trail, configuration compliance, and active threat detection.

### Q16. Instance profile credentials vs static access keys.

Instance profile credentials are temporary, auto-rotated by AWS, and scoped to a role. Static access keys are long-lived secrets that leak easily, never expire unless rotated manually, and require secure storage. In production, no static keys — period. Even CI/CD uses OIDC federation (e.g., GitHub Actions OIDC → IAM role) to get temporary credentials.

### Q17. Private EC2 can't reach the internet.

1. Is there a NAT Gateway in the route table? `0.0.0.0/0 → nat-xxxxx`
2. Is the NAT Gateway healthy and in a public subnet?
3. Is the NAT Gateway's subnet route table pointing `0.0.0.0/0` to the IGW?
4. Security group allows outbound on the required port?
5. NACL allows outbound and inbound (return traffic)?
6. DNS resolution working? Test with `nslookup` or `dig`
7. If access is to an AWS service: consider a VPC Endpoint to avoid NAT entirely

### Q18. Cross-account access for CI/CD.

Use IAM roles with `sts:AssumeRole` and trust policies:
- CI/CD account hosts the build role
- Target accounts have deployment roles with trust policies allowing the CI/CD role to assume them
- Use external IDs to prevent confused deputy attacks
- For GitHub Actions or GitLab CI: configure OIDC federation so the CI runner doesn't need any AWS credentials at all — it exchanges its OIDC token for an AWS role

### Q19. IRSA vs broad node permissions.

Without IRSA, every pod on a node inherits the node's IAM role. If any pod needs S3 access, all pods get it — including a compromised pod. IRSA gives per-Service-Account roles via OIDC, so a pod only gets the AWS permissions it needs. This is least privilege at the workload level, not the node level.

### Q20. Secrets for Kubernetes apps in AWS.

External Secrets Operator + AWS Secrets Manager. The Kubernetes `ExternalSecret` references a secret name. ESO uses an IRSA role to fetch the secret from Secrets Manager and creates a Kubernetes Secret. Secret rotation in Secrets Manager auto-propagates. Never store secrets as plain Kubernetes Secrets in Git or commit AWS credentials anywhere.

### Scenario Q21. Private EC2 can't reach S3.

1. Most likely: no route to S3. Options: NAT Gateway (slow, expensive) or S3 VPC Endpoint (fast, free).
2. Check route table for an S3 Gateway Endpoint
3. If endpoint exists: check endpoint policy and bucket policy — both must allow access
4. Verify IAM role on the instance has `s3:GetObject` for the bucket
5. Test: `aws s3 ls s3://bucket --debug` to see the exact error
6. Common pitfall: bucket region differs from endpoint — endpoints are regional

### Scenario Q22. ALB returning intermittent 502/504.

- **502**: backend connection failure or invalid response — often app crashes or returns malformed HTTP
- **504**: backend timeout — request took longer than ALB's idle timeout (default 60s)

Steps:
1. ALB access logs in S3 — look at `target_status_code` and `request_processing_time`
2. Check target group health — are some targets unhealthy?
3. Application logs at the same timestamps as failed requests
4. Check if it's a specific target (one bad pod/instance) or random
5. Tune ALB idle timeout if requests legitimately take longer
6. For 502: check if the app crashes under load, or if keep-alive timeouts are misconfigured (backend keep-alive must be ≥ ALB idle timeout)

### Scenario Q23. High latency but low CPU.

Bottleneck is elsewhere. Investigate:
1. **Disk I/O**: `iostat` or CloudWatch EBS metrics — high queue depth or latency
2. **Network**: CloudWatch network metrics — packet drops, throughput limits hit
3. **Memory pressure**: swap usage, GC pauses (JVM)
4. **Downstream dependencies**: DB queries, external API calls — measure with APM/tracing
5. **Connection pool exhaustion**: app waiting for free connections
6. **Lock contention**: threads blocked on application or DB locks

Low CPU + high latency almost always means waiting on something, not computing.

### Scenario Q24. ASG not launching new instances.

1. ASG activity history in console — exact failure reason
2. Account or service quota hit (vCPU limit, EIP limit)?
3. Launch template invalid (bad AMI, invalid IAM role, deleted security group)?
4. Insufficient capacity in the AZ for the instance type?
5. Spot instances with no available capacity at the bid price?
6. Health checks failing immediately after launch (instances launched but terminated)?

### Scenario Q25. Multi-account AWS for dev/staging/prod.

I'd use AWS Organizations with this structure:
- **Management account**: only for Organizations and billing — no workloads
- **Security account**: GuardDuty, Config, CloudTrail centralization, audit logs
- **Log archive account**: immutable log storage
- **Shared services account**: shared infrastructure (DNS, shared VPC, CI/CD)
- **Dev / Staging / Prod accounts** per workload domain

Isolation: SCPs at the OU level deny dangerous actions (deleting CloudTrail, disabling GuardDuty). Cross-account access via assume-role only. Network: Transit Gateway for connectivity between accounts. SSO for human access mapped to per-environment roles.

### Scenario Q26. EKS pod needs only one S3 bucket and one secret.

Use IRSA with a tight policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
    },
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:my-app-secret-*"
    }
  ]
}
```

Create the role with an OIDC trust policy locked to the specific namespace and service account, attach this policy, annotate the SA with the role ARN. Done — no other AWS access possible from that pod.

---

## EKS / Kubernetes

### Q1. Kubernetes vs EKS.

Kubernetes is the open-source orchestration platform. EKS is AWS's managed Kubernetes — they run the control plane for you, integrate with AWS IAM, VPC, ELB, EBS. You still operate your own worker nodes (or use Fargate for serverless pods). EKS reduces operational burden of running etcd, API server, scheduler, and controller manager.

### Q2. EKS managed vs your responsibility.

**AWS manages**: control plane HA, etcd, API server, scheduler, controller manager, control plane upgrades, certificate rotation for control plane.

**You manage**: worker nodes (unless Fargate), node OS patching (unless Managed Node Groups with auto-update), node-level configuration, workloads, RBAC, network policies, CNI configuration, ingress controllers, monitoring, logging.

### Q3. Deployment, StatefulSet, DaemonSet, Job.

- **Deployment**: stateless workloads, interchangeable replicas, rolling updates
- **StatefulSet**: ordered, named pods with stable identity and per-pod PVCs (Kafka, ClickHouse, databases)
- **DaemonSet**: one pod per node — log shippers (Fluent Bit), monitoring agents, CNI components
- **Job**: run-to-completion workloads (migrations, batch processing). CronJob for scheduled jobs.

### Q4. Service vs Ingress vs LoadBalancer.

- **ClusterIP Service**: internal-only virtual IP for pod-to-pod traffic within the cluster
- **NodePort Service**: exposes the service on every node's IP at a static port
- **LoadBalancer Service**: provisions a cloud load balancer (ELB/ALB/NLB in AWS) pointing to the service
- **Ingress**: Layer 7 routing — host/path-based rules, TLS termination, managed by an Ingress controller (Nginx, AWS ALB Controller)

For HTTP services, I expose via Ingress backed by one shared ALB. For non-HTTP (MQTT, gRPC streaming), a LoadBalancer service backed by NLB.

### Q5. ConfigMap vs Secret.

- **ConfigMap**: non-sensitive configuration (env vars, config files)
- **Secret**: sensitive data (passwords, tokens, certs) — base64-encoded, not encrypted by default in etcd unless you enable encryption at rest

Both can be mounted as files or env vars. In practice I avoid native Secrets and use External Secrets Operator with AWS Secrets Manager.

### Q6. PodDisruptionBudget.

PDB defines minimum available replicas during voluntary disruptions (node drains, upgrades). Example: `minAvailable: 2` for a 3-replica service means only one pod can be disrupted at a time. Critical for stateful workloads and ensures rolling node upgrades don't take down the service. But be careful — too strict a PDB blocks cluster autoscaler scale-down and node upgrades indefinitely.

### Q7. Requests vs limits.

- **Requests**: guaranteed resources. The scheduler uses this for placement. Pods get at least this much.
- **Limits**: maximum allowed. CPU above limit is throttled. Memory above limit triggers OOMKill.

Best practice: set both. Memory limit ≈ request (memory can't be throttled, only killed). CPU limit can be higher than request to allow bursting. Without requests, the scheduler can't bin-pack effectively. Without memory limits, one pod can starve the whole node.

### Q8. Scheduler decision process.

1. **Filtering**: nodes that can't run the pod are eliminated (insufficient resources, taints not tolerated, node selector mismatch, volume affinity)
2. **Scoring**: remaining nodes are ranked using priorities (least requested, affinity match, topology spread, image locality)
3. Highest-scoring node wins; pod is bound

You can influence this with affinity rules, taints/tolerations, topology spread constraints, and pod priority.

### Q9. Taints, tolerations, node selectors, affinity.

- **Taint**: marks a node to repel pods. `kubectl taint nodes node1 gpu=true:NoSchedule`
- **Toleration**: lets a pod schedule onto a tainted node
- **NodeSelector**: simple label-based node selection
- **Affinity/Anti-affinity**: richer expression — required/preferred, pod-to-node or pod-to-pod

Use taints/tolerations to dedicate nodes (e.g., GPU nodes). Use anti-affinity to spread replicas across nodes/AZs.

### Q10. Readiness vs liveness probes.

- **Readiness probe**: is the pod ready to serve traffic? If failing, removed from Service endpoints. Doesn't restart the pod.
- **Liveness probe**: is the pod alive? If failing, the container is restarted.

Both matter: readiness handles startup time and temporary unhealthiness (e.g., during DB reconnect). Liveness handles stuck processes. A common mistake is using the same endpoint for both — a transient dependency failure shouldn't restart the pod.

### Q11. CrashLoopBackOff.

The container starts, exits with non-zero code, Kubernetes restarts it with exponential backoff (10s, 20s, 40s, up to 5 min). Debug:
1. `kubectl describe pod <pod>` — check exit code
2. `kubectl logs <pod> --previous` — logs from the crashed instance
3. Exit 137 = OOM kill, exit 1 = app error, exit 139 = segfault
4. Check recent ConfigMap/Secret changes
5. Try running the same image locally with the same env

### Q12. Service discovery in Kubernetes.

Pods discover services via DNS. CoreDNS resolves `my-service.my-namespace.svc.cluster.local` to the service's ClusterIP. `kube-proxy` then load-balances to a backing pod. Short names work within a namespace (`my-service`). Each service automatically gets a DNS record on creation.

### Q13. CoreDNS, kube-proxy, CNI plugin.

- **CoreDNS**: in-cluster DNS — resolves service names to ClusterIPs
- **kube-proxy**: implements Service abstraction via iptables/IPVS — routes ClusterIP traffic to actual pods
- **CNI plugin**: provides pod networking — assigns pod IPs, sets up routing between pods across nodes. In EKS, AWS VPC CNI gives each pod a real VPC IP.

### Q14. HPA and limitations.

HPA scales pod replicas based on CPU/memory or custom metrics (every 15s evaluation). Limitations:
- Requires resource requests to calculate utilization percentages
- Slow reaction time — minimum ~30-60s to start scaling
- Can't scale to zero (without KEDA)
- Doesn't help if the bottleneck is downstream (DB, external API)
- Stabilization windows prevent rapid scale-down

### Q15. Cluster Autoscaler and HPA interaction.

HPA increases pod count → if no node has capacity, pods go Pending → Cluster Autoscaler sees Pending pods and adds nodes. CA respects PDBs when scaling down. The risk: if HPA scale-down is too aggressive, nodes drain and CA removes them, then traffic returns and CA must add nodes again (slow). Tune `scale-down-delay-after-add` and HPA stabilization windows to prevent flapping.

### Q16. Securing EKS at all layers.

- **Cluster**: private API endpoint, audit logging, restrict control plane endpoint access
- **Node**: Bottlerocket OS or hardened AMIs, no SSH, IMDSv2 enforced, node IAM role with minimal permissions
- **Pod**: Pod Security Standards (restricted), non-root, read-only root filesystem, dropped capabilities, no privileged
- **Network**: default-deny network policies, namespace isolation, SGs on pods for fine-grained control
- **IAM**: IRSA for per-pod permissions, no shared node roles
- **Images**: ECR with image scanning, admission webhook to block CVEs
- **Secrets**: External Secrets Operator, KMS encryption for Kubernetes secrets at rest in etcd

### Q17. IRSA technical detail.

1. EKS cluster has an OIDC identity provider
2. You create an IAM role with a trust policy that trusts the OIDC provider and restricts to a specific namespace and ServiceAccount
3. You annotate the ServiceAccount with `eks.amazonaws.com/role-arn: <role-arn>`
4. When a pod uses that SA, Kubernetes projects a service account token into the pod at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`
5. The AWS SDK detects the `AWS_WEB_IDENTITY_TOKEN_FILE` env var and calls `sts:AssumeRoleWithWebIdentity` using that token
6. STS validates the token via OIDC, returns temporary credentials scoped to the role

This is all transparent to application code.

### Q18. Safe EKS upgrade.

1. Read release notes — deprecated APIs, breaking changes
2. Upgrade control plane first (AWS handles)
3. Upgrade managed node groups one at a time with surge upgrade settings (`maxSurge: 25%`, `maxUnavailable: 1`)
4. Ensure PDBs exist for critical workloads
5. Test in staging cluster first — same versions, similar workloads
6. Have a rollback plan; EKS doesn't support control plane downgrade, so caution is essential
7. Update kubectl, Helm charts, and any deprecated API usage before upgrading

### Q19. Debugging NotReady node.

1. `kubectl describe node <node>` — check conditions (MemoryPressure, DiskPressure, NetworkUnavailable)
2. SSH or SSM into the node
3. `systemctl status kubelet` and `journalctl -u kubelet -n 200`
4. `systemctl status containerd`
5. `df -h` — disk full? `/var/lib/kubelet` or `/var/lib/containerd`?
6. `free -h` — memory pressure?
7. Network: can the node reach the control plane?
8. If unrecoverable: drain (`kubectl drain --ignore-daemonsets`) and replace the node

### Q20. Network policies in EKS.

EKS uses AWS VPC CNI by default, which doesn't enforce NetworkPolicies. Options:
1. Enable Network Policy feature in VPC CNI (newer versions support it)
2. Install Calico in policy-only mode alongside VPC CNI
3. Replace VPC CNI with Cilium

Start with default-deny in each namespace, then add explicit allow rules:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

### Q21. Multi-tenant EKS.

- **Namespace per team**: isolated resources, RBAC scope
- **ResourceQuotas and LimitRanges**: prevent one team from consuming the whole cluster
- **Network policies**: enforce traffic isolation between namespaces
- **IRSA**: per-team SAs with team-scoped AWS access
- **AppProjects in ArgoCD**: limit which apps each team can deploy
- **Pod Security Standards**: enforce non-privileged workloads
- **Separate node pools** (taints + tolerations) if isolation must extend to the OS level
- **OPA Gatekeeper / Kyverno**: enforce org-wide policies (image registry, required labels)

### Q22. Observing and troubleshooting EKS in production.

- **Metrics**: Prometheus (in-cluster) + Grafana dashboards. CloudWatch Container Insights for AWS-native view.
- **Logs**: Fluent Bit → CloudWatch Logs or Loki
- **Tracing**: OpenTelemetry → Jaeger or AWS X-Ray
- **Audit logs**: Kubernetes audit logs to S3/CloudWatch
- **Events**: monitored via Prometheus event exporter

For troubleshooting: I start at the symptom (latency, errors) and walk back — service metrics → pod logs → node metrics → cluster events.

### Scenario Q23. Pod stuck in Pending.

1. `kubectl describe pod <pod>` — bottom shows the scheduler reason
2. Common causes:
   - **Insufficient resources**: no node has enough CPU/memory for the requests
   - **No matching node selector/taint**: pod requires a label or toleration no node has
   - **PVC not bound**: storage provisioning issue
   - **Image pull errors**: actually shows as `ImagePullBackOff` after scheduling
3. `kubectl get nodes` + `kubectl describe nodes` — check capacity
4. If autoscaler should add nodes: check CA logs and ASG status

### Scenario Q24. Pod restarting every few minutes, incomplete logs.

1. `kubectl logs <pod> --previous` — previous instance logs
2. `kubectl describe pod` — exit code on last termination
3. If memory issue: enable `kubectl top pod` over time
4. Add `terminationMessagePolicy: FallbackToLogsOnError` to capture last log lines
5. Increase log verbosity if app supports it
6. Use `kubectl exec` into a running container to inspect state
7. If liveness probe killing it: check probe config — maybe app is slow to respond under load

### Scenario Q25. HPA scaled up but latency still high.

Scaling pods didn't fix it because the bottleneck is elsewhere:
1. Check downstream dependencies — DB, cache, external API at saturation
2. Connection pool exhaustion on the app side
3. Network/throughput limits on the node
4. Inefficient code path triggered at scale (n+1 queries)
5. CPU throttling on individual pods despite low utilization average (uneven distribution)
6. Look at p99 of individual dependencies — usually one of them is the bottleneck

Throwing more pods at a database problem doesn't help and often makes it worse.

### Scenario Q26. New pods not scheduled despite added nodes.

1. Are the new nodes Ready? `kubectl get nodes`
2. Do the new nodes have taints the pods don't tolerate?
3. Pod affinity rules forcing pods elsewhere?
4. PVC affinity tied to a specific AZ that doesn't match the new nodes' AZ?
5. Node selector mismatch?
6. Resource requests still too large for the new node type?
7. Check scheduler events: `kubectl get events --field-selector reason=FailedScheduling`

### Scenario Q27. Node group upgrade degraded a stateful app.

1. `kubectl get pods -n <ns>` — which replicas are down?
2. Check if PDB was sufficient — likely too permissive, allowing multiple replicas to be drained simultaneously
3. Are PVCs reattaching properly to new nodes?
4. For Kafka/ClickHouse: check cluster membership and ISR status
5. Immediate mitigation: pause the upgrade (`aws eks update-nodegroup-config`), let pods recover
6. Post-incident: tighter PDBs, ensure StatefulSet podManagementPolicy is `OrderedReady` for safe rollouts

### Scenario Q28. Multi-tenant EKS for different business units.

- **IAM**: each BU has its own IAM role, mapped to its namespace via IRSA
- **RBAC**: each BU's group has Role bindings only in its namespace, no ClusterRoles
- **Namespaces**: one per BU (or per app), labeled and annotated for identification
- **ResourceQuotas**: capped CPU, memory, pod count per namespace
- **LimitRanges**: default and max requests/limits per pod
- **NetworkPolicies**: default-deny between namespaces, explicit allow for cross-BU services
- **Cost allocation**: tag resources and namespaces; use Kubecost or AWS Cost Explorer with namespace labels
- **Storage classes**: shared, but quotas enforce per-BU limits

---

## Python

### Q1. Python vs Bash in DevOps.

Bash is great for simple sequential operations and pipes. Python is better for:
- Complex logic, error handling, retries
- Structured data (JSON, YAML)
- API interactions (boto3, requests, kubernetes client)
- Testing and reusability
- Concurrent operations (asyncio, threading)

Rule of thumb: if a script grows past ~50 lines or needs structured data manipulation, I rewrite it in Python.

### Q2. Built-in data types.

- **int, float, complex**: numeric
- **str, bytes**: text/binary
- **list**: ordered, mutable
- **tuple**: ordered, immutable
- **set, frozenset**: unordered, unique elements
- **dict**: key-value mapping
- **bool, NoneType**: boolean and null

Choice depends on mutability, ordering, uniqueness, and access pattern. Use dict for lookup, set for deduplication, tuple for fixed structure.

### Q3. List vs tuple vs set vs dict.

- **List**: mutable, ordered, allows duplicates. `O(n)` lookup. Use for sequences.
- **Tuple**: immutable, ordered, allows duplicates. Hashable, so usable as dict keys. Use for fixed records.
- **Set**: mutable, unordered, unique. `O(1)` lookup. Use for membership tests and deduplication.
- **Dict**: mutable, key-value, ordered (Python 3.7+). `O(1)` lookup. Use for mapping.

### Q4. Exception handling.

```python
try:
    resp = requests.get(url, timeout=5)
    resp.raise_for_status()
except requests.Timeout:
    logger.error("Timeout")
except requests.HTTPError as e:
    logger.error(f"HTTP {e.response.status_code}")
except Exception:
    logger.exception("Unexpected")
    raise
finally:
    cleanup()
```

Catch specific exceptions; use broad `except Exception` only at the top of a script with logging. Always `raise` if you can't handle it.

### Q5. `==` vs `is`.

`==` compares values. `is` compares identity (same object in memory). `None`, `True`, `False` should be checked with `is` because they're singletons: `if x is None`. Strings and numbers should use `==` because small integers and interned strings can have surprising identity behavior.

### Q6. Module vs package.

A module is a single `.py` file. A package is a directory with an `__init__.py` containing multiple modules. Packages can be nested. Both are imported the same way: `import mypackage.mymodule`.

### Q7. Virtual environments.

`venv` (or `uv`, `poetry`, `pipenv`) creates an isolated Python environment with its own dependencies. Critical because:
- Different projects need different package versions
- System Python should not be polluted with project dependencies
- Reproducibility — `pip freeze > requirements.txt` captures exact versions

In production, I always use venv inside containers — the container itself is isolation, but venv keeps dependencies clearly scoped.

### Q8. Reading env vars securely.

```python
import os
db_password = os.environ.get("DB_PASSWORD")
if not db_password:
    raise RuntimeError("DB_PASSWORD env var required")
```

Or use `pydantic-settings` / `python-decouple` for typed config. Never hardcode secrets. Env vars are injected from External Secrets / Secrets Manager / Vault — not from `.env` files committed to Git.

### Q9. `subprocess.run()` vs `os.system()`.

`os.system()` returns only the exit code, spawns a shell, and is vulnerable to injection. `subprocess.run()` gives:
- Captured stdout/stderr
- Timeout support
- Pass arguments as a list (no shell, no injection)
- Custom env, cwd
- `check=True` raises on non-zero exit

```python
result = subprocess.run(
    ["kubectl", "get", "pods", "-n", namespace],
    capture_output=True, text=True, check=True, timeout=30
)
```

Always use `subprocess.run()` for automation.

### Q10. Structuring a maintainable automation script.

- Single responsibility per function
- `if __name__ == "__main__":` entry point
- `argparse` for CLI args
- `logging` not `print`
- Type hints on function signatures
- Constants at module top
- Separate I/O (API calls, file reads) from logic for testability
- Requirements pinned

A maintainable script looks like a small application, not one giant function.

### Q11. Logging in production.

```python
import logging
logging.basicConfig(
    level=os.environ.get("LOG_LEVEL", "INFO"),
    format='{"time":"%(asctime)s","level":"%(levelname)s","msg":"%(message)s","logger":"%(name)s"}',
)
logger = logging.getLogger(__name__)
```

Structured JSON logs so they're parseable by Loki/Elasticsearch. Include trace IDs for distributed tracing. Different log levels for dev vs prod. Never log secrets.

### Q12. Retries, backoff, timeouts.

Use `tenacity`:

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=1, max=30),
    retry=retry_if_exception_type(requests.RequestException)
)
def call_api():
    return requests.get(url, timeout=5)
```

Always set timeouts on every external call. Exponential backoff with jitter. Retry only idempotent operations or transient errors.

### Q13. boto3 example.

```python
import boto3
session = boto3.Session(region_name="us-east-1")
ec2 = session.client("ec2")
paginator = ec2.get_paginator("describe_instances")
for page in paginator.paginate(Filters=[{"Name": "instance-state-name", "Values": ["running"]}]):
    for reservation in page["Reservations"]:
        for instance in reservation["Instances"]:
            print(instance["InstanceId"], instance.get("Tags"))
```

Always use paginators for list operations. boto3 picks up credentials from env, IAM role, or `~/.aws/credentials` automatically.

### Q14. Interacting with Kubernetes from Python.

Use the official `kubernetes` client:

```python
from kubernetes import client, config
config.load_incluster_config()  # or load_kube_config() outside cluster
v1 = client.CoreV1Api()
pods = v1.list_namespaced_pod(namespace="production")
for pod in pods.items:
    print(pod.metadata.name, pod.status.phase)
```

For automation running inside Kubernetes, `load_incluster_config()` uses the pod's ServiceAccount.

### Q15. Sync vs async; when asyncio helps.

Sync code blocks on I/O — one request at a time. Async (asyncio) lets thousands of I/O operations happen concurrently in a single thread. Use asyncio when:
- Many concurrent I/O-bound operations (API calls, DB queries)
- Polling many endpoints
- Building network services

Don't use asyncio for CPU-bound work — use `multiprocessing` instead. Don't mix sync libraries (like `requests`) with async code — use `httpx` or `aiohttp`.

### Q16. Unit testing DevOps scripts.

`pytest` with mocking:

```python
from unittest.mock import patch, MagicMock

def test_list_instances_without_team_tag():
    mock_ec2 = MagicMock()
    mock_ec2.get_paginator.return_value.paginate.return_value = [
        {"Reservations": [{"Instances": [{"InstanceId": "i-1", "Tags": []}]}]}
    ]
    with patch("boto3.client", return_value=mock_ec2):
        result = find_untagged_instances()
    assert "i-1" in result
```

Mock all external calls — AWS, Kubernetes, HTTP. Test the logic, not the SDK. Use fixtures for shared setup. Run tests in CI on every PR.

### Q17. Packaging and deploying a Python tool.

For internal CLI tools:
1. `pyproject.toml` with build system (setuptools, hatch, poetry)
2. `entry_points` for CLI commands
3. Publish to internal PyPI (CodeArtifact, Artifactory) or as a container image
4. Pin all dependencies via `requirements.txt` or lock file
5. For containerized deployment: multi-stage Dockerfile, slim base image, non-root user

For widely-used tools, distribute as a container image to avoid Python version mismatches.

### Q18. Idempotent Python automation.

- Check current state before making changes (`if not exists: create`)
- Use AWS APIs that are idempotent (e.g., `create` operations with explicit IDs)
- Track applied changes (state file, database, tag)
- Make destructive operations require explicit confirmation
- For retries: ensure the same input produces the same result

Idempotency means rerunning the script is safe — critical for automation that may be triggered multiple times.

### Q19. Secrets in Python scripts.

- Read from env vars set by Kubernetes from External Secrets / Secrets Manager
- For local dev: use `direnv` or `.envrc` (gitignored), or fetch from AWS Secrets Manager at runtime
- Never log secret values — `logger.info(f"Using key {key[:4]}...")` if needed
- Use `getpass` for interactive prompts
- For long-running scripts, support credential refresh (STS credentials expire)

### Q20. Observable Python in production.

- **Logs**: structured JSON, shipped to Loki/CloudWatch
- **Metrics**: Prometheus client library, expose `/metrics` endpoint or push to Pushgateway for batch jobs
- **Tracing**: OpenTelemetry SDK, send to Jaeger/X-Ray
- **Errors**: Sentry for exception reporting
- **Health endpoint**: for long-running services, `/healthz` for liveness/readiness

A script you can't observe in production is a script you can't operate.

### Scenario Q21. Daily report of EC2 without required tag.

Design:
1. Lambda function or Kubernetes CronJob running daily
2. Use boto3 with paginators to list all EC2 instances across all regions
3. Filter instances missing the required tag (e.g., `Owner`)
4. Generate a CSV or Markdown report
5. Send to S3 with date-prefixed key + post a summary to Slack
6. Idempotent: each run produces a fresh report; old reports are retained in S3 with lifecycle policy
7. Permissions: IRSA role with `ec2:DescribeInstances`, `s3:PutObject`, no write access to EC2

### Scenario Q22. Works locally, fails in Jenkins.

1. **Python version difference**: local 3.11 vs Jenkins 3.9?
2. **Missing env vars**: AWS credentials, secrets — Jenkins doesn't have your local `.envrc`
3. **Network access**: Jenkins agent in a different VPC, can't reach some endpoint
4. **Working directory differences**: relative paths break
5. **Permissions**: Jenkins user can't write to `/tmp` or specific path
6. **Dependencies not installed in agent**: missing system packages
7. Debug by adding verbose logging, running with `set -x` in shell wrapper, exec into the agent container

### Scenario Q23. Python job getting AWS throttled.

1. boto3 has built-in retries with exponential backoff — increase retry config:
   ```python
   from botocore.config import Config
   config = Config(retries={"max_attempts": 10, "mode": "adaptive"})
   client = boto3.client("ec2", config=config)
   ```
2. Use paginators properly — don't retry the whole pagination on a single failure
3. Add jitter to spread requests
4. Cache results when possible — don't query the same resource repeatedly
5. Run in batches with sleep between batches
6. If still throttled: request a service quota increase

### Scenario Q24. Kafka consumer lag tool across multiple clusters.

Design:
1. Config file lists clusters: bootstrap servers, auth, consumer groups to monitor
2. For each cluster: use `kafka-python` AdminClient to fetch committed offsets per consumer group
3. Fetch latest offsets from each topic-partition
4. Compute lag = latest - committed
5. Threshold per topic in config (different sensitivity per service)
6. Aggregate alerts, dedupe (don't spam Slack every minute)
7. Run as a Kubernetes CronJob every 5 min
8. Expose Prometheus metrics so Grafana dashboards exist independently
9. Send alerts to Slack with severity, lag count, and runbook link
10. Use asyncio if many clusters, to parallelize

### Scenario Q25. Long-running Python script occasionally hangs.

Add observability:
1. **Watchdog**: signal-based timeout that triggers a stack trace dump:
   ```python
   import signal, traceback
   def dump_stack(sig, frame):
       traceback.print_stack(frame)
   signal.signal(signal.SIGALRM, dump_stack)
   signal.alarm(300)
   ```
2. **Heartbeat metric**: increment a Prometheus counter every iteration. If it stops incrementing, alert
3. **py-spy**: attach to a hung process without restarting it: `py-spy dump --pid <pid>`
4. **Set timeouts on every external call** — most hangs are blocked I/O without timeout
5. Log entry/exit of long operations with timestamps
6. If running in Kubernetes: liveness probe based on the heartbeat

### Scenario Q26. Poll 500 pods quickly — threads, multiprocessing, or asyncio?

I'd use **asyncio** because this is pure I/O — waiting on Kubernetes API responses. asyncio scales to thousands of concurrent requests in a single thread with low overhead.

- **Threads**: works but adds GIL contention; fine up to ~50 concurrent
- **Multiprocessing**: overkill for I/O; high memory cost
- **asyncio**: ideal for I/O-bound concurrency at scale

Use `kubernetes_asyncio` or run sync calls in `asyncio.to_thread()` if the client doesn't have native async support.

---

## Ansible

### Q1. Ansible vs shell scripting.

Ansible provides:
- **Idempotency**: modules check state before acting
- **Declarative model**: describe desired state, not steps
- **Inventory management**: hosts and groups
- **Modules for everything**: 1000+ modules for files, services, packages, cloud resources
- **Reusability**: roles, collections
- **Agent-less**: just needs SSH
- **Better error handling**, parallelism, and reporting than shell

Shell is for one-off, simple operations. Ansible is for repeatable configuration across many hosts.

### Q2. Inventory, playbook, role, module, handler.

- **Inventory**: list of hosts and groups (static INI/YAML or dynamic via plugins)
- **Playbook**: YAML file defining what tasks run on which hosts
- **Role**: structured directory of tasks, handlers, variables, templates, files for reusability
- **Module**: the actual unit of work — `copy`, `service`, `yum`, `template`
- **Handler**: a task that runs only when notified, typically at end of play (service restart on config change)

### Q3. Idempotency.

Running the same playbook 10 times produces the same result as running it once. Critical because:
- Configuration drift correction without unintended changes
- Safe to re-run after partial failures
- Predictable, repeatable infrastructure

Ansible modules check current state and only change what's needed. Shell modules (`command`, `shell`) break this unless you explicitly add `creates:` or `when:` guards.

### Q4. `command` vs `shell` vs modules.

- **`command`**: runs a command directly, no shell — no pipes, redirects, env expansion
- **`shell`**: runs through `/bin/sh`, supports pipes and redirects, but vulnerable to injection
- **Native modules**: idempotent, structured, return proper change status

Always prefer native modules. Use `command` for binaries with no module. Use `shell` only when you specifically need shell features. Both `command` and `shell` are non-idempotent by default — add `creates:` or `changed_when:` to make them idempotent.

### Q5. Facts.

Facts are system variables gathered automatically at playbook start — OS, IP, memory, CPU, disk. Access via `ansible_facts.os_family`, `ansible_facts.distribution`. Useful for conditional tasks (`when: ansible_facts.os_family == 'RedHat'`). Disable with `gather_facts: false` to speed up playbooks that don't need them.

### Q6. Variables and precedence.

Variable precedence (low to high):
1. Role defaults (`defaults/main.yml`)
2. Inventory vars
3. Playbook vars
4. Role vars (`vars/main.yml`)
5. Block vars
6. Task vars
7. Extra vars (`-e` on CLI) — highest

The detailed precedence list has 20+ levels. Practical rule: defaults at role level for sane defaults, group_vars for environment-specific overrides, extra vars for one-off CLI overrides.

### Q7. Secrets in Ansible.

- **Ansible Vault**: encrypt files or strings with a passphrase
- **External secret managers**: lookup plugins for AWS Secrets Manager, HashiCorp Vault, Azure Key Vault
- **`no_log: true`** on tasks that handle secrets to keep them out of output

```yaml
- name: Get DB password
  set_fact:
    db_password: "{{ lookup('amazon.aws.aws_secret', 'prod/db/password') }}"
  no_log: true
```

For production, external secret managers are better than Vault because rotation is centralized.

### Q8. Vault vs external secret manager.

- **Vault**: file-based encryption, good for static secrets in Git, simple to start with
- **External secret managers**: dynamic secrets, centralized rotation, audit trail, no passphrase to share

For long-lived infrastructure: external secret manager. For specific encrypted snippets in Git (legacy systems, bootstrap secrets): Vault. Often both: Vault stores the credentials needed to access the external manager.

### Q9. Handlers.

Handlers run only when notified by a task and run once at end of play regardless of how many notifications:

```yaml
tasks:
  - name: Copy nginx config
    template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
    notify: restart nginx
handlers:
  - name: restart nginx
    service: name=nginx state=restarted
```

Even if 5 tasks update nginx config files, nginx restarts once. Without handlers, you'd restart 5 times unnecessarily.

### Q10. Tags.

Tags let you run a subset of tasks:

```bash
ansible-playbook site.yml --tags "nginx,ssl"
ansible-playbook site.yml --skip-tags "debug"
```

Useful in long playbooks for targeted operations — e.g., only rotate SSL certs without re-running the full setup. Tag thoughtfully; over-tagging makes playbooks hard to read.

### Q11. Reusable roles across environments.

- **Defaults** in `defaults/main.yml` for all configurable values
- **Variables** parameterized — no hardcoded paths, IPs, ports
- **Environment-specific values** in `group_vars/<env>/`
- **Document inputs** in `README.md`
- **Version** the role via Ansible Galaxy or Git tags
- **Test** with Molecule
- **Idempotent** — never break on re-run

A good role has zero environment-specific code in the role itself.

### Q12. Jinja2 templates.

Templates use Jinja2 syntax with variables, conditionals, loops:

```jinja
server {
    listen {{ nginx_port }};
    server_name {{ server_name }};
    {% for backend in backends %}
    upstream {{ backend.name }} {
        server {{ backend.host }}:{{ backend.port }};
    }
    {% endfor %}
}
```

Use the `template` module to render — Ansible substitutes variables and copies to target. Always set permissions and use handlers to reload the service on change.

### Q13. AWS dynamic inventory.

Use the `amazon.aws.aws_ec2` inventory plugin:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  instance-state-name: running
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
hostnames:
  - private-ip-address
```

Run `ansible-inventory -i aws_ec2.yaml --list` to verify. The plugin uses boto3 with the same credential chain (IAM role, env vars, profile).

### Q14. Ansible structure for large org.

- **Top-level**: `inventories/` per environment, `playbooks/` for entry points
- **Roles**: shared roles in a `roles/` directory or pulled from Galaxy
- **Collections**: encapsulate related roles + modules into versioned, distributable units
- **group_vars/host_vars**: per-environment variables, with secrets in Vault or external manager
- **CI/CD pipeline**: lint with `ansible-lint`, test with Molecule, gated apply to production
- **Source of truth**: Git; no manual edits on hosts

### Q15. Testing playbooks.

1. **Syntax check**: `ansible-playbook --syntax-check`
2. **Lint**: `ansible-lint` for best practices
3. **Check mode**: `ansible-playbook --check` for dry run
4. **Molecule**: full test framework — spins up containers/VMs, applies role, runs verification (Testinfra)
5. **Staging environment**: run in staging before production
6. **Idempotency test**: run twice; second run should report zero changes

### Q16. Check mode and diff mode.

- **Check mode (`-C`)**: dry run, predicts changes without applying. Not all modules support it perfectly.
- **Diff mode (`-D`)**: shows what would change in files (templates, configs)

Combine them: `ansible-playbook -CD site.yml` shows exactly what would change. Essential before risky changes.

### Q17. Debugging a failing task.

1. **Increase verbosity**: `-v`, `-vv`, `-vvv` for more detail
2. **`debug` module**: print variable values
3. **`--start-at-task`**: skip ahead to the failing task
4. **`--step`**: prompt before each task
5. **`ansible-console`**: interactive debugging on target
6. SSH to the host and inspect actual state
7. Check Ansible logs and `become` permission issues

### Q18. Partial failure across hosts.

By default, Ansible continues on other hosts when one fails — so 18 succeed, 2 fail. Recovery:
1. Identify why 2 failed: connectivity, permission, environment difference
2. Fix the underlying issue
3. Re-run with `--limit failed_hosts` to retry only the failures
4. Use `--retry <file>` if Ansible saved a retry file
5. For critical operations, use `any_errors_fatal: true` to abort the entire play on any failure
6. Use `serial: 1` to run host-by-host for sensitive changes — easier to control blast radius

### Q19. Safe production changes.

- **Rolling execution**: `serial: 5` runs in batches of 5, never all at once
- **`max_fail_percentage: 25`**: abort if more than 25% fail
- **Pre-flight checks**: validate prerequisites before changes
- **Test in staging first**: identical playbook, similar environment
- **Use `check` and `diff` mode**: review predicted changes
- **Backup before changes**: `backup: yes` on file modules
- **Handlers with rolling restarts**: don't restart all services simultaneously
- **Document a rollback procedure**

### Q20. Ansible vs Terraform.

- **Terraform**: provisions infrastructure (VPC, EC2, S3, EKS). Stateful, declarative, knows what exists.
- **Ansible**: configures inside the infrastructure (install packages, edit configs, deploy code). Stateless, procedural-ish.

Use both: Terraform to create the VM, Ansible to configure it. For Kubernetes-era workloads, Terraform creates EKS and Ansible's role shrinks — most config goes into containers and Helm. I still use Ansible for legacy VMs, OS-level configuration, and orchestration of multi-step operations across infrastructure.

### Scenario Q21. Patching 300 servers.

Strategy:
- **Inventory grouped by environment**: dev → staging → prod
- **Serial execution**: `serial: "10%"` to patch 30 at a time
- **Pre-checks**: verify connectivity, disk space, no critical jobs running
- **Drain from load balancer** before patching (if applicable)
- **Apply patches**: `yum update` / `apt upgrade` with security-only flag
- **Reboot if needed**: check `/var/run/reboot-required`, reboot with confirmation
- **Verify after reboot**: services up, health checks passing
- **Re-add to load balancer**
- **`max_fail_percentage: 5`**: abort if too many failures
- **Roll forward environments**: only proceed to prod after staging succeeds

### Scenario Q22. 18 servers configured, 2 inconsistent.

1. **Identify failure cause** on the 2 servers — likely SSH timeout, disk full, or permission issue
2. **Check what got partially applied** — `ansible <failed_host> -m setup` to see state
3. **Fix the underlying issue** (clear disk, fix permissions)
4. **Re-run with `--limit failed_servers`** — playbook should be idempotent and complete the missing tasks
5. **Verify** all 20 are now consistent: a quick read-only Ansible playbook that asserts expected state
6. **Improve playbook** with `block/rescue/always` to handle partial failures more gracefully going forward

### Scenario Q23. Template change requires Nginx restart, but minimize restarts.

Use handlers + serial execution:

```yaml
- hosts: web_servers
  serial: 2  # restart 2 at a time
  tasks:
    - name: Update nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        validate: 'nginx -t -c %s'
      notify: reload nginx
  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded  # graceful, not restart
```

- `validate:` runs `nginx -t` before applying — catches config errors
- `reloaded` (not restarted) = graceful, no connection drop
- `serial: 2` ensures rolling, never all at once
- Handler only fires if template actually changed — no unnecessary reloads

### Scenario Q24. Certificate rotation across servers.

```yaml
- hosts: web_servers
  serial: 1
  tasks:
    - name: Backup current cert
      copy:
        src: /etc/ssl/certs/app.crt
        dest: /etc/ssl/certs/app.crt.bak.{{ ansible_date_time.epoch }}
        remote_src: yes

    - name: Fetch new cert from S3
      amazon.aws.s3_object:
        bucket: my-certs
        object: app.crt
        dest: /etc/ssl/certs/app.crt
        mode: get

    - name: Validate cert
      command: openssl x509 -in /etc/ssl/certs/app.crt -noout -checkend 0
      changed_when: false

    - name: Test nginx config
      command: nginx -t
      changed_when: false

    - name: Reload nginx
      service:
        name: nginx
        state: reloaded

    - name: Verify HTTPS endpoint
      uri:
        url: "https://{{ inventory_hostname }}/health"
        validate_certs: yes
      delegate_to: localhost
```

`serial: 1` ensures one-at-a-time rotation. Validation at every step. If any step fails, the play aborts before propagating to other servers.

### Scenario Q25. Dynamic AWS inventory not returning expected hosts.

1. **Run the plugin directly**: `ansible-inventory -i aws_ec2.yaml --list` and inspect output
2. **Check credentials**: `aws sts get-caller-identity` — does Ansible have the right IAM permissions?
3. **Check filters**: are tag filters too strict? Region missing?
4. **Check instance state**: filter includes only `running`?
5. **Plugin enabled**: `inventory` section in `ansible.cfg` lists `amazon.aws.aws_ec2`
6. **File extension**: must be `*.aws_ec2.yml` or referenced correctly
7. **Verbose mode**: `ANSIBLE_DEBUG=1 ansible-inventory --list -vvv` for detailed errors

### Scenario Q26. Playbook works in staging, fails in production.

Hosts are slightly different — that's the bug to find:
1. **Compare facts**: `ansible <staging_host> -m setup > staging.txt; ansible <prod_host> -m setup > prod.txt; diff staging.txt prod.txt`
2. **Common differences**: OS version, package versions, kernel, disk layout
3. **Make the playbook more defensive**:
   - Use `when:` conditions based on facts
   - Use `package_facts` to check installed versions
   - Add `block/rescue` to handle environment-specific failures
   - Validate prerequisites at the start of the play
4. **Test in a production-like environment** — staging should mirror prod or at least cover the same OS distributions
5. **Document environment assumptions** in the role's README

---

## Cross-Skill End-to-End Scenarios

### Q1. ArgoCD deployment went through, but app down in prod.

Layered investigation:
1. **ArgoCD layer**: app status — Synced + Healthy? Or Synced + Degraded?
2. **Kubernetes layer**:
   - `kubectl get pods` — running? CrashLoop? Pending?
   - `kubectl describe pod` + `kubectl logs --previous`
   - Recent rollout: `kubectl rollout history`
3. **AWS layer**:
   - ALB target health — are targets healthy?
   - Security groups, route tables changed recently?
   - VPC endpoints reachable?
4. **Application layer**:
   - DB connectivity from the pod: `kubectl exec` and test
   - Downstream dependencies — Kafka, ClickHouse, external APIs
   - Recent config changes via External Secrets
5. **Rollback decision**: if it's clearly the new release, `git revert` immediately and investigate root cause separately
6. **Communication**: post in #incidents, declare severity, bring in the right owners

### Q2. Full design — GitOps on EKS with Secrets Manager and least-privilege IAM.

Architecture:
1. **EKS cluster** with OIDC provider enabled
2. **ArgoCD** installed via Helm (bootstrap chart), exposed via internal ALB only
3. **App of Apps**: root app in a platform Git repo deploys all platform components
4. **External Secrets Operator** with IRSA role: `secretsmanager:GetSecretValue` on a tag-scoped resource (`tag:Cluster=prod`)
5. **Per-app SAs**: each app has its own ServiceAccount with IRSA mapped to a tightly-scoped IAM role
6. **AppProjects**: one per team, restricting source repos, target namespaces, and clusters
7. **RBAC**: SSO via Azure AD → ArgoCD RBAC → team-specific permissions
8. **Network policies**: default deny per namespace, explicit allow
9. **Git repos**:
   - `platform/` — root app, platform components
   - `team-x/` — team's apps with environment overlays
10. **Promotion**: PR to update image tag in Git → ArgoCD syncs
11. **Secret rotation**: rotate in Secrets Manager → ESO refreshes Kubernetes Secret → app re-reads or restarts

### Q3. New EKS environment + ArgoCD + Python service + Ansible for legacy VMs.

Responsibility split:
- **Terraform**: VPC, subnets, EKS cluster, node groups, IAM roles, ECR repositories, RDS, Secrets Manager
- **ArgoCD**: deploy platform components (Prometheus, cert-manager, External Secrets) and the Python service via Helm
- **Python service**: containerized, uses IRSA for AWS access, deployed via Helm chart in Git, ArgoCD syncs
- **Ansible**: manages legacy VMs that can't be containerized — OS patching, package installation, config files. Uses dynamic AWS inventory.
- **CI/CD (Jenkins)**: builds Python image → pushes to ECR → updates image tag in Git → ArgoCD picks up

Each tool does what it's best at. No overlap, clear boundaries.

### Q4. Incident: high latency → consumer lag → ArgoCD synced → AWS node pressure.

This is a cascade. Walking the investigation:
1. **Symptom layer**: high latency on EKS service
2. **Confirm scope**: which service? All requests or specific endpoints?
3. **Correlate with consumer lag**: same time window? Is the slow service producing/consuming Kafka messages?
4. **ArgoCD synced means no recent code change** — so it's not a deployment issue. It's runtime or capacity.
5. **AWS node pressure**: CPU? Memory? Disk? Network? Check CloudWatch and Prometheus node-exporter
6. **Likely root cause chain**:
   - Node memory pressure → kubelet evicting pods or pods OOM'd → reduced consumer capacity → Kafka lag → upstream services backed up → latency spike
7. **Immediate actions**:
   - Scale up node group to relieve pressure
   - Scale up consumer pods via KEDA if not already
   - Check if a noisy neighbor pod is consuming the node
8. **Lead the incident**: incident channel, regular updates, document timeline for postmortem
9. **Post-incident**: improve resource requests/limits, add node-level alerts before pressure cascades

### Q5. Security audit found broad IAM access and hardcoded secrets.

Remediation plan:
1. **Audit scope**: enumerate every offending pod and script. Use IAM Access Analyzer to find unused permissions
2. **AWS / IAM**:
   - Replace node IAM roles' overly broad permissions with minimal baseline
   - Implement IRSA for every workload that needs AWS access
   - Each Service Account → dedicated IAM role with least privilege
   - Add SCPs at OU level to deny dangerous actions
3. **ArgoCD / Kubernetes**:
   - Adopt External Secrets Operator
   - Migrate all hardcoded secrets to AWS Secrets Manager
   - Update Helm charts to reference ExternalSecret CRs
   - Rotate every credential that was previously hardcoded
4. **Python tooling**:
   - Replace hardcoded secrets with env vars sourced from K8s Secrets (via ESO)
   - Code review checklist: no string literals matching secret patterns
   - Pre-commit hooks with `detect-secrets` or `gitleaks`
   - Git history scan + rotate any historical secrets
5. **Ansible**:
   - Move plaintext secrets to Ansible Vault or external secret managers
   - Use `no_log: true` on tasks handling secrets
6. **Process**:
   - Mandatory secret scanning in CI for every PR
   - Quarterly IAM access reviews
   - Postmortem on how secrets were committed; fix the gap (training, tooling)

### Q6. When to use ArgoCD vs Ansible vs Python vs AWS services?

My decision framework:
- **AWS native services**: when AWS already provides it well (Lambda, EventBridge, Step Functions, Secrets Manager) and the operational simplicity outweighs vendor lock-in
- **Terraform**: for infrastructure provisioning — anything that creates AWS resources
- **ArgoCD**: for Kubernetes application deployment and lifecycle — declarative, GitOps, drift correction
- **Ansible**: for legacy VM configuration, OS-level changes, multi-step orchestration across non-Kubernetes infrastructure
- **Python**: for custom automation, glue logic, scheduled jobs that need real logic, internal CLIs and tools, anything where the other tools become awkward

The rule: use the most boring, well-supported tool that fits. Don't reinvent in Python what ArgoCD already does. Don't deploy Kubernetes apps via Ansible. Don't write Terraform when a Kubernetes operator already manages that resource. Each tool has a clear lane, and using them outside their lane is where complexity comes from.

---

*End of document.*
