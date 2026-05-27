# Screening Interview: Top 5 Skills (ArgoCD, AWS, EKS, Python, Ansible)

Comprehensive Q&A covering the interviewer's top skills from basic to advanced, including detailed answers and scenarios.

---

## ArgoCD

### Basic to Intermediate Questions

**Q1. What is GitOps, and how does ArgoCD implement it in Kubernetes?**

GitOps means Git is the source of truth for what should be running in the cluster. Instead of people making manual `kubectl` changes, the desired state lives in version-controlled manifests or Helm/Kustomize definitions. ArgoCD continuously compares that desired state in Git with the live cluster state and reconciles differences. In an interview, I would say the real value is auditability, rollback by Git revert, and elimination of configuration drift.

**Q2. What are the core ArgoCD components, and what does each one do?**

The main components are the API server, repo server, and application controller. The API server powers the UI, CLI, RBAC, and incoming requests. The repo server pulls Git content and renders Helm, Kustomize, or plain manifests. The application controller does the actual reconciliation by comparing desired and live state and applying changes. In many setups, Dex is used for SSO and Redis is used for caching.

**Q3. What is the difference between `Refresh`, `Sync`, `Auto-Sync`, `Self-Heal`, and `Prune`?**

`Refresh` updates ArgoCD's view of Git and cluster state but does not apply changes. `Sync` applies the desired state to the cluster. `Auto-Sync` means ArgoCD will sync automatically when it detects drift. `Self-Heal` means it can also correct drift caused by manual cluster changes, not just Git changes. `Prune` removes resources that exist in the cluster but no longer exist in Git, which is important for avoiding orphaned resources.

**Q4. What is the difference between desired state, live state, and target state in ArgoCD?**

Desired state is what is defined in Git. Live state is what is actually running in the cluster. Target state is often used to mean the rendered manifests ArgoCD intends to apply after processing Helm or Kustomize. I usually explain it like this: Git tells us what should exist, the cluster shows what does exist, and ArgoCD computes the rendered target it will use to make the two match.

**Q5. How do you onboard a new application into ArgoCD from scratch?**

I start by making sure the application's deployment manifests are Git-friendly and environment-aware, usually via Helm or Kustomize. Then I define an ArgoCD `Application`, point it to the repo path, branch, and namespace, and attach it to the correct ArgoCD `Project` for RBAC boundaries. After that, I verify repo access, cluster access, sync policy, and health checks. In a mature setup, I avoid manual app creation and use `ApplicationSet` or an App-of-Apps pattern so onboarding itself is automated.

**Q6. What is the `App of Apps` pattern, and when would you use it?**

The App-of-Apps pattern means one parent ArgoCD application manages multiple child ArgoCD applications. I use it when I want a cluster bootstrap model where a single root app deploys platform services, namespaces, ingress, observability, and workload apps. It is especially useful for multi-team or multi-cluster environments because it gives a single entry point for cluster composition while keeping each application independently managed.

**Q7. What is an `ApplicationSet`, and how is it different from a normal `Application`?**

A normal `Application` represents one deployment target. An `ApplicationSet` is a generator that creates many ArgoCD applications from a template. For example, I can generate one application per cluster, one per environment, or one per repo folder. If I need to deploy the same app to dozens of clusters or namespaces, `ApplicationSet` is the right tool because it avoids manually creating and maintaining a large number of nearly identical `Application` objects.

**Q8. How do you structure Git repositories for ArgoCD in a multi-team or multi-environment setup?**

I prefer separating application source from deployment configuration. A common model is one repo for application code and another repo for environment manifests or Helm values. For example, the app repo builds the image, while the config repo controls what image tag runs in dev, staging, and prod. This keeps promotion PR-driven and makes production access safer because teams can build code without necessarily having direct write access to prod configuration.

**Q9. How do you manage environment-specific values for Helm applications deployed through ArgoCD?**

I keep a base chart and provide separate value files per environment, such as `values-dev.yaml`, `values-staging.yaml`, and `values-prod.yaml`. ArgoCD can point to the same chart path but use different value files per application definition. I avoid copy-pasting full charts per environment because that becomes hard to maintain. The key is to keep environment differences explicit but minimal, with most logic remaining shared.

**Q10. How does ArgoCD handle Helm, Kustomize, and plain manifests differently?**

ArgoCD does not behave exactly like `helm install` as a release manager. It uses Helm mainly as a templating engine, renders the YAML, and then applies the manifests. With Kustomize, it builds overlays into final YAML before applying. With plain manifests, it applies them directly. I explain this carefully because many people think ArgoCD is "running Helm" in the same way Jenkins or a human would, but it is really reconciling rendered Kubernetes resources.

**Q11. How do sync waves and hooks work in ArgoCD? Give a real use case for `PreSync` or `PostSync`.**

Sync waves let me control ordering across resources by assigning phases or weights, which is helpful when dependencies matter. Hooks let me run resources at certain lifecycle stages like `PreSync`, `Sync`, or `PostSync`. A good example is a `PreSync` database migration job that must succeed before the new application version comes up. A `PostSync` hook is useful for smoke tests or notifying a deployment channel after rollout.

**Q12. How do you manage secrets in a GitOps workflow without storing plaintext secrets in Git?**

I prefer external secret systems such as AWS Secrets Manager, HashiCorp Vault, or Azure Key Vault, combined with External Secrets Operator or a CSI driver. Another common option is Sealed Secrets or SOPS-encrypted files. My principle is simple: Git can hold encrypted data or secret references, but not raw secrets. That keeps GitOps safe without breaking automation.

**Q13. How do you implement RBAC in ArgoCD so teams can only manage their own applications?**

I use ArgoCD `Projects` as the first layer of isolation. A project can restrict which clusters, namespaces, and repositories a team can use. Then I map identity provider groups to project roles through ArgoCD RBAC policy. That way a team can sync, view, and manage only the applications inside its own project and cannot accidentally affect platform or another team's workloads.

**Q14. How do you integrate ArgoCD with SSO or enterprise identity providers?**

Most commonly through OIDC or SAML, often using Dex as an identity broker or directly integrating with Okta, Azure AD, or another IdP. I configure group claims and then map those groups into ArgoCD RBAC. The important operational point is not just login, but consistent group-based authorization. I avoid individual user-based permissions because they do not scale and are harder to audit.

**Q15. What does it mean when an application is `OutOfSync` but still `Healthy`?**

That means the live resources differ from Git, but the workload itself is still functioning correctly from Kubernetes' perspective. For example, someone may have manually changed a replica count or a controller may have mutated a field. The app is running, so health is fine, but ArgoCD sees drift. In an interview I would say this is a very common operational case and usually means I need to determine whether the drift is acceptable, expected, or something ArgoCD should correct.

**Q16. What does it mean when an application is `Synced` but `Degraded`?**

That means ArgoCD successfully applied the desired manifests, but the resulting workload is unhealthy. For example, the Deployment object matches Git, but the pods are crashing or never become ready. This is where people sometimes misunderstand GitOps: being synced only means configuration convergence, not application correctness. At that point I move from ArgoCD into Kubernetes-level troubleshooting.

**Q17. How do you debug an ArgoCD application stuck in `Progressing` state?**

I start by identifying which resource is still not healthy. Then I check Kubernetes events, rollout status, pod logs, job status, PVC binding, and admission webhook behavior. If the application uses hooks, I inspect whether a `PreSync` or `PostSync` job is blocked. My goal is to narrow the problem from "ArgoCD is progressing" to "this specific Deployment, Job, or StatefulSet is preventing health from becoming green."

**Q18. How do you safely roll back a failed ArgoCD deployment?**

The cleanest rollback is usually a Git revert or a promotion reversal, because it preserves Git as the source of truth. ArgoCD will then sync the previous desired state back into the cluster. In urgent cases, I may pause auto-sync temporarily, recover service, and then make Git reflect the emergency action immediately afterward. I avoid long-lived manual divergence because it defeats GitOps.

**Q19. What is drift detection in ArgoCD, and how do you handle manual changes done directly in the cluster?**

Drift detection means ArgoCD recognizes when the live state no longer matches the desired state in Git. If `Self-Heal` is enabled, it can automatically revert those changes. Operationally, I treat manual changes as emergency-only. If someone changes the cluster during an incident, the real fix is to backport that change into Git or explicitly decide it should be ignored via ArgoCD diff customization, not let the cluster remain different from source control.

**Q20. How would you implement promotion from dev to staging to production using ArgoCD?**

I prefer environment-specific config in Git and promote by pull request. For example, dev may use image tag `1.4.3-rc`, and after validation a PR updates staging, then another PR updates prod. That keeps promotion auditable, reviewable, and reproducible. I do not like ad hoc "click deploy to prod" models in GitOps because they break the traceability that GitOps is supposed to provide.

### ArgoCD Scenarios

**Scenario 1. A production application is constantly going `OutOfSync` every few minutes even though nobody is changing Git. How do you investigate?**

My first suspicion is a field being changed by another controller, webhook, or Kubernetes defaulting behavior. I compare the live and desired manifests carefully and look for fields like `replicas`, annotations injected by sidecars, webhook-mutated security context, or status-like data mistakenly tracked. If the drift is expected and harmless, I configure `ignoreDifferences`. If it is not expected, I identify the other actor causing drift and decide whether ArgoCD or that controller should own the field.

**Scenario 2. ArgoCD shows `Synced`, but the new pods are crashing and traffic is failing. What do you do next?**

At that point the problem is no longer ArgoCD reconciliation, it is workload behavior. I inspect the Deployment rollout, pod events, readiness probe failures, logs, config changes, image tag, and dependent services like databases or queues. I also check whether the new config introduced a bad environment variable or missing secret. If impact is high, I revert in Git first to restore service, then investigate the exact defect.

**Scenario 3. A `PreSync` migration job is failing, so the application never deploys. How do you debug and recover?**

I inspect the migration job logs, events, container image, runtime environment, and database connectivity. Then I determine whether the failure is because of the migration logic itself, permissions, bad secret injection, or an environmental dependency. Recovery depends on the cause: if the migration is wrong, I fix it in Git; if it partially applied, I assess whether the migration is idempotent or needs manual remediation. I never force the app past a failed migration unless I fully understand the schema state.

**Scenario 4. You need to deploy the same application to 40 clusters with minor differences per cluster. How would you design that in ArgoCD?**

I would use `ApplicationSet`, most likely with a cluster generator or matrix generator. The application template would be common, and per-cluster differences would be expressed through values or overlays, not duplicated manifests. I would keep only the genuinely different values per cluster, such as domain name, region, or storage class, while everything else stays shared. This keeps the fleet consistent and makes upgrades realistic.

**Scenario 5. A developer manually changed a Kubernetes resource in production to fix an outage, and ArgoCD reverted it automatically. How would you handle this situation operationally?**

I would treat it as a process issue, not just a technical one. In an outage, manual changes sometimes happen, but the on-call team must either pause auto-sync briefly or immediately reflect the emergency change in Git. After the incident, I would review whether the team needs a documented emergency procedure for GitOps-managed environments. The real lesson is that if Git is authoritative, incident workflows have to respect that.

---

## AWS

### Basic to Intermediate Questions

**Q1. What are the main differences between Security Groups and NACLs?**

Security Groups are stateful and attached to ENIs or instances, so return traffic is automatically allowed. NACLs are stateless and attached to subnets, so I must define both inbound and outbound rules explicitly. In practice, I use Security Groups for most day-to-day access control and NACLs for broader subnet-level guardrails. If I had to choose one to explain clearly in a screening call, I would say Security Groups are the primary control and NACLs are the coarse-grained secondary layer.

**Q2. What is the difference between an IAM user, role, group, and policy?**

An IAM user is a long-lived identity, usually for humans or legacy tooling, though I try to minimize their use. A group is a way to attach permissions to multiple users. A role is an assumable identity with temporary credentials, which is what I prefer for workloads and cross-account access. A policy is the JSON document that defines permissions. The key design principle is to prefer roles over static users whenever possible.

**Q3. How do IAM roles work for EC2 and EKS workloads?**

For EC2, a role is attached through an instance profile, and the workload gets temporary credentials from the metadata service. For EKS, the best practice is IRSA, where a Kubernetes service account is mapped to an IAM role and the pod assumes that role via web identity. That gives pod-level permission boundaries instead of node-level permission sprawl. In an interview, I would emphasize that temporary, scoped credentials are both safer and easier to rotate than static keys.

**Q4. What is a VPC, and what are the core building blocks inside it?**

A VPC is the logically isolated network boundary in AWS. Its core building blocks are subnets, route tables, internet gateways, NAT gateways, security groups, NACLs, and DNS settings. Depending on the design, I also include VPC endpoints, peering, Transit Gateway, and network ACL strategies. I explain it as the base network foundation on which all compute and managed services sit.

**Q5. What is the difference between public and private subnets?**

A public subnet has a route to an Internet Gateway, so resources can have direct internet reachability if they also have a public IP or load balancer fronting them. A private subnet does not route directly to the internet. If workloads in a private subnet need outbound internet access, they usually go through a NAT Gateway or VPC endpoint. I put most application and database workloads in private subnets and only expose what needs to be internet-facing.

**Q6. What is the role of an Internet Gateway, NAT Gateway, and route table?**

The Internet Gateway allows traffic between a VPC and the internet for public subnets. The NAT Gateway lets private subnets initiate outbound internet access without becoming directly reachable from the internet. The route table determines where packets go, such as to a local VPC route, IGW, NAT, or VPC endpoint. I usually explain AWS networking as a combination of placement plus routing plus security.

**Q7. What is the difference between ALB, NLB, and Classic Load Balancer?**

ALB is Layer 7, so it understands HTTP and HTTPS and supports path-based or host-based routing. NLB is Layer 4 and is best for TCP/UDP, low latency, and preserving client IP, which makes it a good fit for protocols like MQTT. Classic Load Balancer is the older generation and I generally avoid it for new designs. In modern architectures, it is almost always ALB or NLB depending on protocol needs.

**Q8. When would you choose Route 53 weighted routing, latency routing, or failover routing?**

Weighted routing is useful for traffic splitting, controlled migrations, or simple canary strategies. Latency routing is useful when I want users to reach the region with the lowest network latency. Failover routing is for active-passive patterns where one endpoint is primary and another is standby. I choose based on whether my goal is experimentation, performance optimization, or disaster recovery.

**Q9. What is the difference between S3, EBS, and EFS?**

S3 is object storage and is ideal for artifacts, backups, logs, and static content. EBS is block storage attached to an EC2 instance, typically used for databases or instance-local persistent disks. EFS is a managed shared file system that can be mounted by multiple instances. I usually explain it as object, block, and file storage, each optimized for different access patterns.

**Q10. What is the difference between RDS and self-managed databases on EC2?**

RDS reduces operational burden because AWS manages backups, patching, failover, and much of the underlying maintenance. Self-managed databases on EC2 give more control but also require me to handle OS hardening, backup design, replication setup, upgrades, and monitoring myself. I choose RDS unless I have a strong technical reason not to, such as special extensions, storage patterns, or software not supported by the managed offering.

**Q11. How do Auto Scaling Groups work, and what metrics can drive scaling?**

An ASG maintains a desired number of EC2 instances and can scale based on metrics, schedules, or custom alarms. Common metrics include CPU, request count, or queue depth depending on the workload. The ASG can also replace unhealthy instances automatically. I describe it as the compute elasticity layer for EC2-based architectures.

**Q12. What is CloudWatch, and how do you use metrics, logs, dashboards, and alarms together?**

CloudWatch is the observability backbone for AWS-native telemetry. Metrics tell me numerical behavior over time, logs capture event-level detail, dashboards give an at-a-glance operational view, and alarms turn thresholds or anomaly detection into action. In production I use them together, not separately. Metrics tell me something is wrong, logs help explain why, and alarms ensure the issue is noticed.

**Q13. How do you design a highly available application in AWS across multiple AZs?**

I spread compute across at least two or preferably three AZs, put stateless services behind a load balancer, and use a database tier with multi-AZ or application-level replication. I also design health checks, scaling, and failover paths so one AZ loss does not become a full application outage. High availability is not just about multi-AZ placement; it also depends on how state, dependencies, and recovery behaviors are designed.

**Q14. How do you secure an AWS account for production workloads?**

I start with strong identity controls: SSO, MFA, least privilege, no root usage except account setup, and no long-lived keys where avoidable. Then I enable CloudTrail, Config, GuardDuty, security alerting, log retention, and account-level guardrails through Organizations and SCPs if applicable. At the workload layer, I enforce encryption, secret management, private networking, and least-privilege IAM roles. The point is defense in depth, not one control.

**Q15. What is the purpose of CloudTrail, Config, and GuardDuty?**

CloudTrail records API activity and gives me an audit history of who did what. Config tracks resource configuration and helps identify drift or noncompliance. GuardDuty analyzes signals and detects suspicious behavior such as credential misuse or unusual network activity. Together they give auditability, compliance visibility, and threat detection.

**Q16. What is the difference between instance profile credentials and static access keys?**

Instance profile credentials are temporary and automatically rotated by AWS. Static access keys are long-lived secrets that need manual handling and are risky if leaked. I prefer instance profiles or IRSA because they remove secret distribution from the equation. In a security-conscious environment, static keys should be the exception, not the default.

**Q17. How would you troubleshoot connectivity from a private EC2 instance to the internet?**

I would check the subnet's route table, confirm there is a default route to a NAT Gateway, verify the NAT is in a public subnet with an Internet Gateway, and inspect security groups and NACLs for outbound and return path rules. Then I would test DNS resolution and actual egress from the instance. If only certain destinations fail, I also consider proxy settings, VPC endpoints, or endpoint-specific policies.

**Q18. How do you design cross-account access for CI/CD or platform teams?**

I use IAM roles with trust policies that allow specific principals from a source account to assume them. The CI/CD system authenticates in its own account and assumes least-privilege roles in target accounts such as dev, staging, or prod. I prefer this over duplicating users across accounts. If the environment is mature, I also combine this with AWS Organizations, SCPs, and dedicated log archive or security accounts.

**Q19. What is IRSA in EKS, and why is it better than attaching broad permissions to worker nodes?**

IRSA stands for IAM Roles for Service Accounts. It lets a specific pod or workload assume a dedicated IAM role through a Kubernetes service account instead of inheriting whatever permissions the node has. That means a pod can have access to only one S3 bucket or one secret instead of the whole node having broad permissions. This is one of the most important EKS security patterns.

**Q20. How do you handle secrets securely in AWS for applications running in Kubernetes?**

I store the secrets in AWS Secrets Manager or Parameter Store and sync them into Kubernetes through External Secrets Operator or a CSI driver. Access is controlled through IRSA so only the intended workload can read the secret. I avoid checking secrets into Git and avoid storing long-lived credentials in plain Kubernetes secrets unless I have a very controlled reason and encryption-at-rest is clearly addressed.

### AWS Scenarios

**Scenario 1. An EC2 instance in a private subnet cannot reach S3. Walk me through the debugging steps.**

I first determine whether the design expects NAT-based egress or an S3 VPC endpoint. If it uses NAT, I verify the route table, NAT health, subnet association, SGs, and NACLs. If it uses an S3 endpoint, I check the endpoint policy, route table associations, DNS resolution, and bucket policy. I also confirm whether the problem is network reachability or IAM authorization, because "cannot reach S3" often blends connectivity and permission issues together.

**Scenario 2. Your application behind an ALB is returning intermittent `502` or `504` errors. What do you check first?**

I start with the target group health and the ALB access logs or metrics. A `502` often points to the backend closing the connection unexpectedly or invalid responses, while `504` often points to backend slowness or timeout mismatch. I inspect application response times, health checks, ALB idle timeout, backend keepalive settings, and whether a downstream dependency is making the app slow. Intermittent errors usually mean the system is partially healthy, which makes timing and dependency analysis important.

**Scenario 3. A production workload is experiencing high latency, but CPU utilization on the EC2 instances is low. How do you investigate?**

Low CPU tells me the bottleneck is likely not raw compute. I check memory pressure, disk I/O wait, network latency, database response times, lock contention, connection pool saturation, and external dependency slowness. I also look at thread pools or event loops because apps can be blocked even when CPU is idle. In interviews, I emphasize that low CPU does not mean low load; it often means the system is waiting.

**Scenario 4. An Auto Scaling Group is not launching new instances even though demand has increased. How would you debug it?**

I check whether the scaling policy actually triggered, whether the ASG max size has already been reached, and whether launch template or launch configuration settings are valid. Then I look for subnet IP exhaustion, instance type capacity shortages, service quotas, suspended scaling processes, or health check issues. In many real cases, it is either a configuration limit or capacity availability problem, not the scaling alarm itself.

**Scenario 5. You need to design a secure multi-account AWS setup for dev, staging, and prod. How would you structure it and why?**

I would separate prod from non-prod at minimum, and ideally use additional shared accounts such as identity, logging, and security. This reduces blast radius, simplifies access control, and makes billing and governance cleaner. I would put guardrails in place with Organizations and SCPs, centralize CloudTrail and security findings, and use cross-account roles for platform and CI/CD workflows. The main reason is isolation, especially for prod.

**Scenario 6. An EKS pod needs access to one S3 bucket and one Secrets Manager secret only. How would you implement least-privilege access?**

I would create an IAM policy granting only the exact `s3` actions on that bucket and the exact `secretsmanager:GetSecretValue` access on that one secret ARN. Then I would attach that policy to a dedicated IAM role and map it to the pod's Kubernetes service account via IRSA. That service account would be used only by that workload. This gives pod-level least privilege and avoids leaking permissions to other workloads on the same node.

---

## EKS / Kubernetes

### Basic to Intermediate Questions

**Q1. What is the difference between Kubernetes and EKS?**

Kubernetes is the open-source orchestration platform itself. EKS is AWS's managed Kubernetes service. With EKS, AWS manages the control plane for me, but I still operate node groups, networking design, add-ons, security, storage integration, and workload reliability. So EKS reduces control-plane burden, but it does not remove operational responsibility.

**Q2. Which parts of the control plane are managed by AWS in EKS, and which parts are still your responsibility?**

AWS manages the managed control plane components such as the API server and etcd availability. I am still responsible for worker nodes or Fargate profiles, CNI behavior, add-on versions, RBAC, IAM integration, application manifests, security posture, cost, and observability. I usually explain this as: AWS owns the brain availability, but I still own most of the platform and workload operations.

**Q3. What is the difference between a Deployment, StatefulSet, DaemonSet, and Job?**

A Deployment is for stateless replicated applications. A StatefulSet is for workloads needing stable identity and persistent storage per replica. A DaemonSet ensures one pod per node or per matching node, which is useful for log shippers or security agents. A Job runs to completion and is useful for one-time or batch tasks. I choose the workload controller based on lifecycle and state semantics, not habit.

**Q4. What is the difference between a Service, Ingress, and LoadBalancer service type?**

A Service provides stable networking for a set of pods. A `LoadBalancer` service asks the cloud provider to provision an external load balancer, usually for direct exposure. An Ingress is an HTTP/HTTPS routing layer that can route to multiple services based on hostnames and paths. I think of Service as the internal abstraction and Ingress as the external HTTP entry layer.

**Q5. What is a ConfigMap and what is a Secret? When would you use each?**

A ConfigMap stores non-sensitive configuration such as flags, URLs, or text files. A Secret is meant for sensitive data like passwords, API tokens, or keys. I still remind interviewers that Kubernetes Secrets are only base64-encoded by default and should be paired with encryption-at-rest and good access controls. Operationally, I use external secret stores for high-sensitivity environments.

**Q6. What is a PodDisruptionBudget, and why is it important in production?**

A PDB defines how many replicas must remain available during voluntary disruptions like node drains or rolling upgrades. Without it, an upgrade or autoscaler action can evict too many pods at once and create avoidable downtime. In production, it is especially important for stateful or quorum-based systems where too many simultaneous disruptions can break service or consensus.

**Q7. What is the difference between requests and limits in Kubernetes?**

Requests affect scheduling because they tell the scheduler the minimum resources a pod needs. Limits cap how much the container can consume. CPU can be throttled when it exceeds the limit, and memory can result in OOM kills if the process goes beyond the limit. I explain this carefully because bad requests distort scheduling and bad limits cause instability.

**Q8. How does the Kubernetes scheduler decide where to place pods?**

The scheduler first filters nodes that do not satisfy hard requirements such as resource availability, taints, affinity, or storage constraints. Then it scores the remaining nodes based on factors like spreading, balance, and preferences. The final placement is not random; it is a decision based on feasibility and scoring. That is why pod scheduling problems are usually diagnosable through scheduler events and placement rules.

**Q9. What are taints, tolerations, node selectors, and affinity rules?**

Taints repel pods from nodes unless the pod has a matching toleration. Node selectors and node affinity attract pods toward nodes with certain labels. Affinity and anti-affinity can be used to co-locate or spread pods relative to nodes or other pods. I use these tools to control placement intentionally, for example keeping system workloads on dedicated nodes or keeping replicas spread across failure domains.

**Q10. How do readiness probes and liveness probes differ? Why do both matter?**

Readiness tells Kubernetes whether a pod is ready to receive traffic. Liveness tells Kubernetes whether the container is still functioning and should be restarted if not. A pod can be alive but not ready, such as during startup or degraded dependency conditions. I treat readiness as traffic safety and liveness as recovery safety.

**Q11. What happens when a pod goes into `CrashLoopBackOff`?**

It means the container starts, fails, and Kubernetes keeps retrying with backoff. The key is not the backoff itself but the reason for the repeated crash. I check `kubectl describe pod`, `kubectl logs --previous`, exit codes, events, and whether it is OOM, bad config, missing dependency, or startup bug. The status is a symptom, not a diagnosis.

**Q12. How does service discovery work inside Kubernetes?**

Kubernetes creates DNS records for services, usually through CoreDNS. Pods can reach a service by its service name and namespace, and the Service routes traffic to matching pod endpoints. This abstracts away pod IP churn. In practice, I rely heavily on service discovery because pods are ephemeral and direct pod addressing is not stable.

**Q13. What is the role of CoreDNS, kube-proxy, and the CNI plugin in EKS?**

CoreDNS handles internal DNS resolution. `kube-proxy` programs the rules that implement Service networking. The CNI plugin assigns network identities to pods and integrates pod networking with the underlying infrastructure. In EKS, understanding these pieces is important because DNS failure, service routing issues, and IP exhaustion can all look like "the app is broken" even when the app is fine.

**Q14. What is HPA, and what are its limitations?**

HPA scales pod replicas based on metrics such as CPU, memory, or custom metrics. It is useful, but it is reactive and metric-driven, not predictive. It depends on good resource requests and on the right metric choice. If the real bottleneck is queue lag, external dependency latency, or startup time, HPA alone may not solve the problem.

**Q15. What is Cluster Autoscaler, and how does it interact with HPA?**

HPA decides how many pods I want. Cluster Autoscaler decides whether there are enough nodes to run those pods. If HPA increases replicas and pods remain pending, Cluster Autoscaler may add nodes. The two work at different layers, but they must be tuned carefully so they do not flap or create long recovery delays.

**Q16. How do you secure EKS at the cluster, node, pod, and IAM levels?**

At the cluster level, I secure access to the API server, enable audit logging, and control RBAC. At the node level, I use hardened AMIs, patching, least-privilege instance roles, and restricted SSH or no SSH. At the pod level, I use security contexts, non-root containers, network policies, and admission controls. At the IAM level, I use IRSA and least privilege. The important thing is that EKS security is layered; there is no single switch for it.

**Q17. What is IRSA, and how does it work technically inside EKS?**

IRSA uses the EKS cluster's OIDC provider and a Kubernetes service account token projected into the pod. The pod presents that token to AWS STS via `AssumeRoleWithWebIdentity`, and AWS returns temporary credentials for the IAM role mapped to that service account. This is powerful because the credential scope is tied to the service account identity, not to the node. I usually say it gives AWS-native least privilege to Kubernetes-native identities.

**Q18. How do you manage EKS upgrades safely?**

I upgrade in stages: verify version compatibility, test in lower environments, update cluster add-ons, then upgrade the control plane and node groups carefully. I also validate PDBs, autoscaling, and application readiness behavior before touching production. Managed node groups make this easier, but workload safety still depends on how the applications are written and scheduled.

**Q19. How do you debug a node that shows `NotReady`?**

I start by describing the node and checking its conditions like memory pressure, disk pressure, and network availability. Then I inspect kubelet logs, container runtime health, and whether the node can still reach the control plane. I also check disk usage, especially image and container storage paths. In practice, `NotReady` is often caused by kubelet failure, storage exhaustion, or network problems.

**Q20. How do you implement network policies in EKS, and what do you need from the CNI to support them?**

NetworkPolicy resources only work if the underlying network plugin supports enforcement. Depending on the setup, that may mean using a network-policy-capable CNI or enabling the relevant feature set in the platform. Once support exists, I typically start with deny-by-default and then explicitly allow namespace-to-namespace, DNS, and platform traffic. The biggest mistake is assuming policy YAML alone is enough without checking actual enforcement capability.

**Q21. How do you design multi-tenant EKS clusters so teams are isolated from each other?**

I use namespaces as the basic boundary, combined with RBAC, ResourceQuotas, LimitRanges, network policies, and separate IRSA roles per workload. I also use admission policies and clear ownership of namespaces. That said, for strong regulatory or trust boundaries, I still consider separate clusters because namespace isolation is good, but it is not equivalent to full infrastructure isolation.

**Q22. How do you observe and troubleshoot EKS workloads in production?**

I combine cluster events, pod logs, metrics, traces where available, and cloud-level signals such as node health or load balancer metrics. Good observability usually includes Prometheus and Grafana for metrics, log aggregation, control plane logging, and clear dashboards for saturation, errors, and latency. Troubleshooting is much faster when the team can move from symptom to component quickly rather than relying on a single tool.

### EKS Scenarios

**Scenario 1. A pod is stuck in `Pending` state. How do you systematically debug it?**

I begin with `kubectl describe pod` because the scheduler events usually tell me why it is pending. Common reasons are insufficient CPU or memory, taints without tolerations, node selector or affinity mismatch, PVC issues, or max-pod/IP exhaustion. I then confirm whether there are actually suitable nodes and whether autoscaling is working. The key is that `Pending` is almost always a scheduling or infrastructure fit problem, not an application bug.

**Scenario 2. A production pod is restarting every few minutes, but logs are incomplete. What commands and checks do you use?**

I use `kubectl logs --previous` to get logs from the prior container instance and `kubectl describe pod` for events and exit reasons. I inspect readiness and liveness probe failures, OOM kills, node events, and recent config or secret changes. If needed, I also look at kubelet or runtime logs on the node. Incomplete logs often happen because the process exits too fast, so `--previous` is critical.

**Scenario 3. HPA increased replicas, but response times are still high and users are impacted. What could be happening?**

The first possibility is that the bottleneck is not CPU or the metric HPA uses. The new pods may also take too long to become ready, or the real bottleneck may be downstream like a database, queue, or external API. It is also possible the pod requests are wrong, causing noisy scheduling or throttling. My point in the interview would be that "more pods" only helps if the workload is horizontally scalable and the limiting resource is actually addressed by scaling.

**Scenario 4. New pods are not getting scheduled even though you added nodes. What cluster-level issues could explain this?**

The nodes may not yet be ready, may lack required labels, may carry taints the pod does not tolerate, or may be in the wrong AZ for attached storage. The cluster may also have IP exhaustion or max-pods-per-node limits. I also check whether pod affinity/anti-affinity rules or topology spread constraints are too restrictive. Adding nodes is not enough if the new nodes still do not satisfy the pod's placement constraints.

**Scenario 5. A node group upgrade started and now a stateful application is degraded. How do you investigate and recover?**

I check whether PDBs were respected, whether too many replicas drained at once, whether PVCs reattached correctly, and whether the pods came up in the right order. For stateful systems, I also check quorum, replication lag, and storage availability by AZ. Recovery might mean pausing the upgrade, restoring healthy replicas, or rolling back application-level change if the issue is not the nodes themselves. Stateful upgrades require much more caution than stateless rolling updates.

**Scenario 6. You need to run workloads from different business units in the same EKS cluster. How would you handle IAM, RBAC, namespaces, quotas, and network isolation?**

I would create separate namespaces per business unit, use group-based Kubernetes RBAC, and give each workload its own IRSA role rather than sharing node permissions. I would enforce ResourceQuotas and LimitRanges to keep one unit from exhausting cluster resources. For traffic isolation, I would implement NetworkPolicies with deny-by-default. I would also define clear admission and labeling standards so operational boundaries remain visible. If the business units have materially different trust levels, I would still raise the question of separate clusters.

---

## Python

### Basic to Intermediate Questions

**Q1. Why do you use Python in DevOps or SRE work instead of only Bash?**

Bash is great for simple command chaining and quick shell automation. Python is better when the logic becomes non-trivial, such as retries, structured error handling, API integrations, data processing, testing, packaging, and reuse. I use Bash when the job is basically shell orchestration, but if I am building something I expect to maintain, share, or extend, I strongly prefer Python.

**Q2. What are the main built-in data types in Python, and when do you choose one over another?**

The main ones I use most are strings, integers, floats, booleans, lists, tuples, sets, and dictionaries. Lists are ordered and mutable, tuples are ordered and immutable, sets are useful for uniqueness and fast membership checks, and dictionaries map keys to values. In interviews, I explain choice in terms of behavior: mutability, uniqueness, ordering, and lookup patterns.

**Q3. What is the difference between a list, tuple, set, and dictionary?**

A list is ordered and mutable, so it works well for collections I want to modify. A tuple is ordered and immutable, so it is good for fixed records. A set is unordered and stores unique items only, which is useful for de-duplication. A dictionary stores key-value pairs, which makes it ideal for structured or labeled data. I generally choose the simplest structure that matches the need.

**Q4. How does exception handling work in Python?**

Python uses `try`, `except`, `else`, and `finally`. I catch specific exceptions when I know how to respond, log context clearly, and avoid broad exception swallowing because it hides real bugs. For automation, proper exception handling is important because failures should be explicit, actionable, and ideally recoverable when appropriate. I treat exception strategy as part of the design, not an afterthought.

**Q5. What is the difference between `==` and `is`?**

`==` checks value equality, while `is` checks object identity. Two different objects can be equal in value but not be the same object in memory. I use `is` mostly for identity checks like `value is None`, not for normal string or list comparisons. This is a common screening question because it reveals whether the person understands Python semantics or only writes by pattern.

**Q6. What is the difference between a module and a package?**

A module is a single Python file containing code. A package is a directory of modules, often with an `__init__.py`, used to organize larger codebases. I think of modules as files and packages as structured namespaces. This matters when moving from one-off scripts into maintainable internal tools.

**Q7. What is a virtual environment, and why is it important?**

A virtual environment isolates Python dependencies for a project so that package versions do not clash across tools or the system Python. It improves reproducibility and avoids "works on my machine" dependency issues. In CI/CD and shared teams, I consider virtual environments or containerized runtimes essential for predictability.

**Q8. How do you read environment variables securely in Python applications?**

I read them with `os.environ` or `os.getenv`, validate required values explicitly, and never print secrets into logs. The security part is less about the Python call itself and more about how those values are provided, such as CI secrets, Kubernetes secrets, or external secret injection. I also fail fast if required environment variables are missing, rather than continuing in a partially configured state.

**Q9. What is the difference between `subprocess.run()` and `os.system()`?**

`subprocess.run()` gives me structured control over arguments, exit codes, stdout, stderr, timeouts, and shell usage. `os.system()` is much more primitive and harder to use safely. In production automation I almost always use `subprocess.run()` because it is safer, more testable, and better for error handling.

**Q10. How do you structure a small automation script so it is maintainable and testable?**

I separate input handling, business logic, and side effects. That means parsing arguments cleanly, keeping core logic in functions, and isolating external calls such as AWS, shell commands, or HTTP requests behind small wrappers. Then I add logging, timeouts, and a `main()` entry point. Even for "small" scripts, this structure matters because today's 50-line script often becomes tomorrow's operational tool.

**Q11. How do you log properly in Python for production automation?**

I use the `logging` module rather than raw prints. I include structured context such as resource IDs, region, action, and result, and I choose appropriate levels like INFO, WARNING, and ERROR. I avoid logging secrets, and I make logs useful for debugging retries or failures later. Good automation logs should explain what the script attempted, what happened, and what the operator should know next.

**Q12. How do you handle retries, backoff, and timeouts in Python when calling external systems?**

I set explicit timeouts on every network call and implement bounded retries with exponential backoff, especially for transient failures such as throttling or temporary network issues. I also distinguish retryable errors from permanent errors. Blind retries can make outages worse, so I prefer controlled retry policies with logging and metrics. Libraries like `tenacity` are useful if used thoughtfully.

**Q13. How would you interact with AWS from Python using `boto3`?**

I use `boto3` clients or resources, depending on the service and preferred style. I make sure the runtime gets credentials from a safe source like instance profiles or IRSA, not hardcoded keys. I handle pagination, throttling, and region scoping explicitly. In interviews I often mention that many production bugs in AWS automation come from not handling pagination or retries properly.

**Q14. How would you interact with Kubernetes from Python?**

I can use the official Kubernetes Python client. I load config either from `kubeconfig` locally or in-cluster service account config when the tool runs inside Kubernetes. Then I interact with resources through the appropriate API groups, such as `CoreV1Api` or `AppsV1Api`. For operational tools, I am careful with watch streams, pagination, and permissions.

**Q15. What is the difference between synchronous and asynchronous Python, and when would `asyncio` help?**

Synchronous code does one thing at a time, which is simple and often enough. `asyncio` helps when the work is I/O-bound and I need to manage many concurrent operations efficiently, such as polling large numbers of network endpoints or Kubernetes resources. It is not automatically better; it adds complexity. I use it when concurrency is clearly needed and the libraries support it cleanly.

**Q16. How do you write unit tests for a DevOps automation script?**

I design the script so the core logic is separate from external systems. Then I unit test the logic with mocks for AWS, filesystem, HTTP, or shell calls. I usually use `pytest` because it is concise and readable. Good tests check behavior such as correct filtering, retry handling, and error paths, not just the happy path.

**Q17. How do you package and deploy a Python-based internal tool used by multiple teams?**

I package it as a proper Python project with dependency pinning, tests, and a CLI entry point. Then I distribute it either as a Python package, a container image, or both, depending on how teams consume it. For reliability, I version releases and keep backward-compatible interfaces when possible. The goal is to make usage consistent and upgrades predictable.

**Q18. How would you make a Python automation safe to rerun multiple times?**

I make it idempotent by checking current state before changing anything and by using upsert-style logic rather than assuming a blank slate. For example, if I create a resource, I first look up whether it already exists. If I post alerts or write outputs, I avoid duplicating them unnecessarily. In operations, rerun safety matters because automation often gets retried after partial failures.

**Q19. How do you handle secrets in Python scripts without hardcoding them?**

I retrieve them from environment injection, secret managers, or workload identity-based access to cloud secret stores. I never hardcode them in source code or config repos. I also ensure logs, exceptions, and debug output do not leak them. Secret handling is as much about output discipline and runtime controls as it is about input source.

**Q20. How do you make Python scripts observable in production, with logs, metrics, and error reporting?**

I add structured logging, exit codes, and if the tool is important enough, I emit metrics such as success count, failure count, latency, and retry rate. I also report exceptions to a central log or error-tracking system. If it is a scheduled job, I make sure there is visibility into missed runs or abnormal run duration. I want operators to know not just whether it failed, but how it behaved over time.

### Python Scenarios

**Scenario 1. Write the design for a Python script that finds all EC2 instances without a required tag and reports them daily.**

I would design it as a scheduled job, probably running in Lambda, ECS, or a Kubernetes CronJob depending on the environment. The script would enumerate regions, paginate through EC2 instances, normalize tags into dictionaries, filter for missing required tags, and generate a report. I would send the report to Slack, email, or a ticketing system and include account, region, instance ID, and owner metadata if available. I would also make the tag requirements configurable, so the tool is reusable instead of hardcoded.

**Scenario 2. You wrote a Python script that works locally but fails in Jenkins. How do you debug the difference?**

I compare runtime inputs first: Python version, environment variables, credentials, working directory, dependency versions, and network access. Then I add verbose logging around configuration loading and external calls. Jenkins failures are often caused by missing environment, different IAM context, or dependency drift. My approach is to make the environment assumptions explicit rather than guessing.

**Scenario 3. A Python job that calls multiple AWS APIs is getting throttled. How do you handle that gracefully?**

I would confirm whether the issue is concurrency, too-frequent polling, or missing pagination control. Then I would add exponential backoff with jitter, reduce unnecessary calls, batch when possible, and cache repeated lookups. I also ensure the SDK's retry behavior is understood and not fighting my own retry logic. Good throttle handling means being respectful of the API, not just waiting longer after being rate-limited.

**Scenario 4. You need a Python tool that checks Kafka consumer lag across multiple clusters and posts alerts to Slack. How would you design it?**

I would separate the design into three parts: cluster connectors, lag calculation, and alerting/reporting. The tool would iterate over configured clusters, query group offsets and end offsets, compute lag by group/topic/partition, then aggregate results into alert conditions. Slack notifications should be deduplicated and severity-based, not spam every run. I would also make thresholds configurable and add clear logging so operators can tell whether the issue is connectivity, auth, or actual lag growth.

**Scenario 5. A long-running Python script occasionally hangs in production. What would you add to detect and troubleshoot the issue?**

I would add timeouts around every external dependency, heartbeat logging, and metrics on current step and elapsed duration. If it is multithreaded or async, I would also inspect deadlock or await behavior. For deep debugging, I might add stack dump capabilities or use signal-triggered trace output. The first goal is to make the hang visible; the second is to capture where it got stuck.

**Scenario 6. You need to poll 500 Kubernetes pods and collect health information quickly. Would you use threads, multiprocessing, or asyncio, and why?**

If the work is mostly network I/O and I have async-compatible clients, I would choose `asyncio`. If async support is weak or complexity is not justified, a thread pool is a good practical choice because network polling is I/O-bound, not CPU-bound. I would not choose multiprocessing unless I had heavy CPU work to do. In interviews, I usually say the decision should follow the workload, not fashion.

---

## Ansible

### Basic to Intermediate Questions

**Q1. What problem does Ansible solve, and how is it different from shell scripting?**

Ansible gives me declarative, idempotent configuration management and orchestration across many machines. Shell scripting is procedural and often assumes a starting state, which makes it more fragile at scale. Ansible modules know how to check current state and change only what is needed. That makes repeat runs safer and results more predictable.

**Q2. What is the difference between an inventory, playbook, role, module, and handler?**

Inventory tells Ansible which hosts exist and how they are grouped. A playbook is the high-level workflow that applies tasks to those hosts. A role is a reusable structure for organizing tasks, templates, handlers, defaults, and files. A module is the unit of work, like managing a package or service. A handler is a special task that runs only when notified, typically for restarts or reloads.

**Q3. What is idempotency, and why is it critical in configuration management?**

Idempotency means I can run the same automation repeatedly and end in the same desired state without causing unnecessary changes. This is critical because infrastructure automation gets rerun after failures, during audits, and during incremental rollout. Without idempotency, re-running automation can create drift, downtime, or hard-to-debug side effects.

**Q4. What is the difference between `command`, `shell`, and proper Ansible modules?**

`command` runs a command directly without a shell. `shell` runs through a shell and is more flexible but riskier for quoting, idempotency, and security. Proper Ansible modules should be preferred because they understand state and often support check mode and cleaner outputs. I use shell only when there is no better module and I document why.

**Q5. What are facts in Ansible, and when would you gather or disable them?**

Facts are system information Ansible collects from target hosts, such as OS family, interfaces, CPU, and memory. They are useful for conditional logic and platform-specific tasks. I disable fact gathering when it is unnecessary and I want faster runs, especially in large fleets. In production, unnecessary facts can waste time across hundreds of servers.

**Q6. How do variables work in Ansible, and what is variable precedence?**

Variables can come from defaults, role vars, inventory, group vars, host vars, extra vars, and more. Variable precedence decides which value wins if the same variable appears in multiple places. I keep the design simple: low-priority defaults in roles, environment- or host-specific values in inventory, and `--extra-vars` only for intentional operator overrides. Good variable hygiene prevents confusing behavior.

**Q7. How do you manage secrets securely with Ansible?**

I either encrypt them with Ansible Vault or fetch them from an external secret manager at runtime. I never keep plaintext secrets in repo files. I also avoid printing secret values in debug output and use `no_log: true` where needed. Secret management is not just storage; it includes making sure execution output does not leak sensitive data.

**Q8. What is Ansible Vault, and when would you use it versus an external secret manager?**

Ansible Vault encrypts variables or files inside the repository, which is useful when the team wants Git-contained automation with controlled decryption. An external secret manager is better when I want centralized rotation, access audit, and cross-tool secret reuse. I use Vault when the team is smaller or the use case is contained. I prefer external managers when the environment is larger or has stronger security and audit requirements.

**Q9. How do handlers work, and why are they important?**

Handlers are tasks triggered only when notified by a changed task. For example, a config template updates a file and notifies a service restart handler. This avoids unnecessary restarts when nothing changed. In production, that matters because unnecessary restarts cause risk, especially across large fleets.

**Q10. What are tags in Ansible, and how do they help during operations?**

Tags let me run only certain parts of a playbook, such as only patching, only config deployment, or only validation. They are useful during staged operations or incident recovery because I do not always want the whole playbook. Tags improve operator control, but they should be used thoughtfully so playbooks do not become fragmented or hard to understand.

**Q11. How do you make an Ansible role reusable across multiple environments?**

I keep the role logic generic and parameterize environment-specific values through defaults, vars, and inventory. I avoid embedding environment names or hardcoded paths directly in tasks. I also provide sane defaults and clear documentation for required variables. A reusable role should be adaptable without editing the role itself.

**Q12. How do you manage templates and configuration files with Jinja2 in Ansible?**

I use Jinja2 templates for config files that need environment-specific or host-specific values. The template task renders the file and, if it changes, notifies the appropriate handler. I am careful to keep templates readable and not overload them with too much logic. The best templates are mostly configuration with light variable substitution.

**Q13. How do you work with dynamic inventory in AWS?**

I use the AWS dynamic inventory plugin so Ansible discovers hosts based on AWS APIs rather than a hand-maintained static list. That lets me group hosts by tags, region, VPC, or instance metadata. It is especially useful in autoscaling or ephemeral environments where server inventories change regularly. Dynamic inventory makes Ansible fit cloud infrastructure much better.

**Q14. How do you structure Ansible for a large organization with many teams and services?**

I separate shared roles, environment-specific inventory, and service-specific playbooks. I standardize naming, tagging, and common role interfaces so different teams can reuse the same patterns. I also put guardrails around production execution, such as approval workflows, check mode in CI, and canary runs. The goal is not just to automate, but to keep the automation governable.

**Q15. How do you test Ansible playbooks before running them in production?**

I use syntax checks, linting, and where possible Molecule or ephemeral test environments. I also use `--check` and `--diff` to preview behavior, though I know not every module fully supports check mode. For critical changes, I test on a canary host or small group first. In operations, confidence comes from layered validation, not one tool.

**Q16. What is the purpose of `check mode` and `diff mode`?**

Check mode simulates what Ansible would do without making changes, as long as the modules support it. Diff mode shows what file or config changes would occur. Together they make automation reviewable before execution. I treat them as valuable preview tools, not perfect substitutes for real testing.

**Q17. How do you debug a failing Ansible task in production?**

I increase verbosity, inspect the exact error output, confirm the target host state, and check whether facts, variables, permissions, or connectivity differ from expectations. I also look at whether the task used the right module or fell back to shell unnecessarily. My approach is always to identify whether the issue is data, environment, or task logic.

**Q18. How do you handle partial failures when a playbook succeeds on some hosts and fails on others?**

First I determine whether the completed hosts are safe to leave as-is or whether the fleet now needs reconciliation. Then I fix the root cause and rerun only on failed hosts if that is safe. For more sensitive workflows, I use `serial`, `max_fail_percentage`, prechecks, and rollback logic where applicable. Partial success is not unusual in real infrastructure, so the playbook design should anticipate it.

**Q19. How do you make Ansible runs safe for critical production changes?**

I use staged rollout with `serial`, pre-validation tasks, backups where appropriate, idempotent modules, and post-change health checks. I also make handlers conditional and avoid broad shell commands that are hard to reason about. For especially critical systems, I prefer canarying the change on a small subset before touching the full fleet. Safe automation is usually slower, but much more reliable.

**Q20. When would you choose Ansible over Terraform, and when would you use both together?**

I use Terraform for infrastructure provisioning and stateful cloud resources like VPCs, clusters, or databases. I use Ansible for configuration management, package installation, file templating, and operational orchestration on existing hosts. Together they work well: Terraform creates the infrastructure, and Ansible configures what runs on it. I choose the tool based on what owns the desired state: infrastructure objects or host configuration.

### Ansible Scenarios

**Scenario 1. You need to patch 300 Linux servers across environments with minimal downtime. How would you design the Ansible execution strategy?**

I would use a rolling strategy with `serial`, likely starting with a small canary batch. I would include prechecks, package update steps, reboot handling if needed, and post-patch health validation before moving to the next batch. I would also segment by environment so prod is last after validation in lower tiers. The key is controlled rollout, not blasting all 300 hosts at once.

**Scenario 2. An Ansible playbook updated config on 18 servers and failed on 2, leaving the fleet inconsistent. What do you do next?**

I first assess whether the 18 updated servers are healthy. Then I determine why the two failed, whether due to host drift, permissions, disk space, or something else. Once the cause is clear, I rerun safely on the failed servers or, if necessary, roll back the successful ones if consistency is critical. The right choice depends on whether the new config is safe to run partially.

**Scenario 3. A template change requires Nginx restart, but you want to avoid unnecessary restarts across the fleet. How do you design that?**

I would deploy the template with the `template` module and notify a handler only when the rendered file actually changes. The handler would test Nginx config before reload or restart. That way unchanged hosts do nothing, and changed hosts are handled safely. This is a classic example of why Ansible handlers are so useful.

**Scenario 4. You need to rotate certificates across all Linux servers and verify the service after each change. How would you implement it in Ansible?**

I would do it in a rolling fashion, likely `serial: 1` or a small batch size, depending on service redundancy. The play would back up current certs, copy the new ones, validate permissions, run config validation, reload the service, and then perform an application-level health check. If a host fails validation, I would restore the backup on that host before proceeding. Certificate rotation should be cautious because a bad cert or bad chain can cause immediate outage.

**Scenario 5. Your dynamic AWS inventory is not returning the expected hosts. How do you debug it?**

I verify AWS credentials and region scoping first, then inspect the inventory plugin config, filters, tags, and cache behavior. I also run `ansible-inventory --list` to see exactly what Ansible thinks exists. Many times the issue is either filter mismatch or the host metadata not matching the grouping assumptions. I also confirm whether the expected instances are actually in the account and region I am querying.

**Scenario 6. A playbook works in staging but fails in production because some hosts are slightly different. How do you make the automation more robust?**

I identify what is truly variable across those hosts and encode that through facts, conditionals, or inventory data instead of assuming uniformity. I add preflight checks so the playbook fails early with a clear reason rather than half-applying changes. Over time, I also try to reduce unmanaged drift because production-only host differences are usually a warning sign. Good automation is explicit about environmental variation instead of discovering it accidentally.

---

## Cross-Skill End-to-End Scenarios

**Scenario 1. You are using ArgoCD to deploy applications on EKS. A deployment went through, but the application is down in production. Walk me through how you would debug this across ArgoCD, Kubernetes, AWS, and the application layer.**

I would start by separating reconciliation success from application success. In ArgoCD, I would confirm what commit or chart version was deployed and whether the app is `Synced` or `Degraded`. In Kubernetes, I would inspect rollout status, pod readiness, events, logs, secret mounts, and config changes. In AWS, I would check the load balancer target health, node health, security group changes, and any dependency like RDS or S3. At the application layer, I would look at recent config flags, database migrations, and error rates. My method is to narrow the issue one boundary at a time instead of treating "deployment failed" as a single opaque problem.

**Scenario 2. A team wants GitOps on EKS with ArgoCD, secrets from AWS Secrets Manager, and least-privilege IAM. Design the full solution end to end.**

I would run EKS with ArgoCD managing application manifests from Git. Secrets would live in AWS Secrets Manager and be pulled into the cluster using External Secrets Operator or a CSI-based solution. Each workload would have its own Kubernetes service account mapped to a dedicated IAM role via IRSA, granting only the exact secret and cloud resource permissions it needs. ArgoCD would manage the ExternalSecret definitions, not the raw secret values. RBAC and ArgoCD Projects would limit team access. This gives Git-driven deployment, externalized secrets, and least-privilege cloud access without hardcoded credentials.

**Scenario 3. You need to bootstrap a new EKS environment in AWS, configure ArgoCD, deploy a Python service, and use Ansible for legacy VM configuration. How would you divide responsibility across tools?**

I would use Terraform or cloud-native IaC to provision AWS infrastructure and the EKS cluster. ArgoCD would manage cluster-native applications and continuous delivery on Kubernetes. The Python service would be packaged into a container and deployed through ArgoCD-managed manifests or Helm. Ansible would handle legacy VM configuration, package management, or services that are not yet on Kubernetes. The dividing principle is simple: infrastructure with IaC, Kubernetes delivery with GitOps, application logic in code, and host configuration with Ansible.

**Scenario 4. An incident starts with high latency on EKS, then consumer lag increases, ArgoCD shows everything synced, and AWS metrics show node pressure. How do you lead the investigation?**

I would declare an incident and keep the team aligned on timeline and hypotheses. Since ArgoCD is synced, I would treat deployment drift as less likely and focus on runtime behavior. High latency plus consumer lag plus node pressure suggests resource saturation or scheduling stress, so I would inspect node CPU, memory, disk, pod throttling, pending pods, and whether autoscaling responded correctly. I would also check whether the bottleneck is at Kafka consumers, a downstream database, or insufficient pod distribution across nodes. The key is not to get distracted by GitOps status when the evidence points to runtime capacity and dependency behavior.

**Scenario 5. A security audit found that several pods in EKS have too-broad AWS access and some operational scripts contain embedded secrets. What remediation plan would you propose across AWS, ArgoCD, Python tooling, and Ansible?**

First, I would inventory which workloads have excessive access and which scripts contain secrets. Then I would replace broad node IAM permissions with IRSA-based per-service-account roles and least-privilege policies. I would move embedded secrets into AWS Secrets Manager or another approved secret store and update Python tools and Ansible playbooks to retrieve them securely at runtime. In ArgoCD, I would ensure Git contains only secret references or encrypted objects, not plaintext. Finally, I would add preventative controls: code scanning for secrets, policy checks for IAM scope, and review gates so the problem does not recur.

**Scenario 6. Your senior interviewer asks: "How do you decide when to solve a problem with ArgoCD, Ansible, Python, or native AWS services?" How would you answer that clearly?**

I would say I choose the tool based on what kind of state I am managing and what operational model I want. If I am managing Kubernetes desired state continuously from Git, ArgoCD is the right choice. If I am configuring hosts or coordinating OS-level changes on servers, Ansible fits best. If I need custom logic, data processing, or API-driven automation, Python is ideal. If AWS already provides a managed service that solves the problem well, I prefer the native service rather than building unnecessary glue. My general rule is to choose the highest-level reliable abstraction that solves the problem cleanly.

---

## Interview Tips

- **Listen carefully**: The interviewer may interrupt or dig deeper. Pause and clarify if needed.
- **Be honest about gaps**: If you do not know something, say so. Then pivot to what you do know.
- **Use examples from real work**: Scenarios resonate more than textbook definitions.
- **Show systems thinking**: Explain interactions between components, not just individual tools.
- **Emphasize production readiness**: Production is different. Show you think about reliability, safety, and observability.
- **Ask clarifying questions**: "Are we assuming this is a new environment or an existing one?" helps you answer more relevantly.

Good luck with your screening call!
