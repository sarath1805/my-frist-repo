# DevOps Interview Full Answers

This document consolidates the complete interview answers covering the common skills between your resume and the JD.

---

## 1. Linux

### Q1. What is the difference between a process and a thread in Linux?

A process is an independent program in execution with its own memory space, file descriptors, and PID. A thread is a lightweight unit of execution within a process. Threads share the same memory space and resources of the parent process. Context switching between threads is cheaper than between processes. In practice, when I look at something like EMQX or Kafka brokers, they are heavily multi-threaded, and multiple threads share heap memory, which is why a memory leak in one thread can crash the whole process.

### Q2. How do you check memory and CPU usage on a Linux system?

For CPU and memory at a glance, I use `top` or `htop`. For a point-in-time snapshot, `vmstat 1 5` gives me CPU, memory, swap, and I/O together. For memory specifically, `free -h` shows used vs available vs cached. For per-process details, `ps aux --sort=-%mem | head -20`. For I/O wait issues, `iostat -x 1` and `iotop`. On production nodes I usually start with `top`, look at load average vs CPU count, then drill into specific PIDs.

### Q3. Explain file permissions. What does `chmod 755` mean?

In octal notation, 7 is the owner permission set and means read, write, and execute. 5 is group and means read and execute. The last 5 is others and also means read and execute. So the owner can do everything, while group and others can only read and execute. This is typical for binaries and scripts that everyone should be able to run, but only the owner should modify. I use `chmod 600` for private keys and `chmod 640` for config files with sensitive values.

### Q4. How does `systemd` manage services? How do you troubleshoot a failed service?

`systemd` uses unit files. Service units define how a process starts, stops, and restarts. It handles dependency ordering through directives like `After=`, `Requires=`, and `Wants=`. To troubleshoot a failed service I start with `systemctl status <service>` because it shows the last few log lines and the exit code. Then I use `journalctl -u <service> -n 100 --no-pager` for more detailed logs. If it looks like a crash, I check the `ExecStart` command in the unit file, verify the binary path and environment variables, and if needed run the same binary manually as the same user to reproduce the issue.

### Q5. What is the difference between a hard link and a soft link?

A hard link is another directory entry pointing to the same inode, so it is effectively the same file with multiple names. Deleting one hard link does not delete the underlying data as long as another hard link still exists. A soft link, or symbolic link, is a pointer to a path. It can cross filesystems and point to directories, but if the target is deleted, the symlink becomes broken. In operations work, I use symlinks often for config management, for example linking enabled Nginx configs to available configs.

### Q6. How do you find which process is using a specific port?

I use `ss -tulnp | grep <port>` or `lsof -i :<port>`. `ss` is the modern replacement for `netstat` and is usually faster. This gives the PID and process name directly. On Kubernetes nodes, this is useful for debugging port conflicts or verifying that a service is actually listening before moving higher in the stack.

### Q7. Explain Linux kernel namespaces and cgroups. How do they relate to containers?

Namespaces provide isolation, so each container gets its own view of key system resources. These include PID, network, mount, UTS, IPC, user, and cgroup namespaces. Cgroups are used for resource control, such as CPU, memory, and I/O limits. Containers are essentially Linux processes running inside a combination of namespaces with cgroup limits applied. Docker and containerd rely directly on these kernel primitives. This understanding is important when debugging OOM kills, because the kernel acts on cgroup limits rather than host-wide assumptions.

### Q8. How would you tune kernel parameters to improve network throughput?

The parameters I commonly tune through `sysctl` include:

- `net.core.somaxconn` to increase the TCP backlog queue
- `net.ipv4.tcp_rmem` and `net.ipv4.tcp_wmem` to increase TCP buffer sizes
- `net.core.netdev_max_backlog` to increase the packet receive queue
- `net.ipv4.tcp_tw_reuse` to reuse TIME_WAIT sockets more efficiently
- `vm.swappiness=10` to reduce swap usage on application servers

These matter especially for high-throughput brokers and APIs. We applied similar tuning on EMQX nodes and saw connection establishment times improve.

### Scenario. A production Linux server shows load average 45 on an 8-core machine, but CPU is only 20 percent. Walk me through your diagnosis.

High load with low CPU usually means the bottleneck is not compute, but waiting. My steps would be:

1. Run `iostat -x 1` to check for high I/O wait and disk latency.
2. Run `iotop` to find which process is doing heavy disk I/O.
3. Run `vmstat 1` and check the blocked process count.
4. Verify whether a network filesystem or NFS mount is involved.
5. Check `dmesg` for disk or hardware errors.
6. If the workload involves a database like ClickHouse or PostgreSQL, inspect slow query logs.

In many cases this turns out to be storage wait, a large background merge, or log rotation on a full disk.

---

## 2. AWS

### Q1. What is the difference between Security Groups and NACLs in AWS?

Security Groups are stateful, instance-level firewalls. If inbound traffic is allowed, the return traffic is automatically allowed. NACLs are stateless, subnet-level firewalls, so you must explicitly allow both inbound and outbound traffic. I use Security Groups as the main access control mechanism, and NACLs as an additional coarse-grained control when needed.

### Q2. Explain the difference between an IAM Role and an IAM Policy.

A Policy is a document that defines permissions, meaning which actions are allowed on which resources. A Role is an identity that policies are attached to, and that identity can be assumed by AWS services, EC2 instances, Lambda functions, or pods through IRSA. Roles rely on temporary credentials through STS, which is more secure than using long-lived access keys.

### Q3. What is the difference between S3 Standard, Infrequent Access, and Glacier?

S3 Standard is for frequently accessed data and offers low latency with higher storage cost. Standard-IA is cheaper for storage but charges retrieval fees, which makes it suitable for backups or less frequently accessed objects. Glacier is for archival data with retrieval times measured in minutes or hours and very low storage cost. I often use lifecycle policies to move old data automatically between these classes.

### Q4. How does VPC peering work? What are its limitations compared to Transit Gateway?

VPC Peering creates a direct one-to-one connection between two VPCs. It does not support transitive routing. That means if A is peered with B and B is peered with C, A cannot reach C through B. Transit Gateway solves this with a hub-and-spoke model and supports transitive routing, making it better for larger multi-account or multi-team topologies.

### Q5. How does IAM Roles for Service Accounts (IRSA) work in EKS?

IRSA maps a Kubernetes Service Account to an IAM Role using OIDC federation. The EKS cluster is associated with an OIDC identity provider. The Service Account is annotated with the IAM Role ARN. Pods using that Service Account receive a projected web identity token, and the AWS SDK exchanges that token for temporary credentials. This gives fine-grained AWS access to pods without overprivileging the worker nodes.

### Q6. What is the difference between ALB and NLB? When would you choose one over the other?

ALB works at Layer 7 and understands HTTP and HTTPS. It supports path-based routing, host-based routing, and WebSockets. NLB works at Layer 4 and is built for TCP or UDP with very low latency and high connection volume. I choose ALB for HTTP microservices and NLB for protocols like MQTT that require persistent TCP connections and minimal overhead.

### Q7. How do you design a multi-AZ, multi-region failover strategy for a stateful application on EKS?

Within a region, I spread node groups across multiple AZs, use topology spread constraints, and ensure storage is replicated or handled at the application layer. Across regions, I use Route 53 health checks and failover or latency-based routing, combined with replicated stateful backends such as Kafka replication, ClickHouse replicas, or database replication. The final design depends on RPO and RTO requirements.

### Q8. Explain how you would secure an EKS cluster.

I secure EKS in layers:

- Private API endpoint where possible
- Control plane audit logging to CloudWatch
- Strict Kubernetes RBAC with least privilege
- IRSA for pod-level AWS access
- Pod security controls such as non-root, read-only root filesystem, and no privileged containers
- Network policies for east-west traffic restrictions
- Image scanning and admission control
- External secret management through AWS Secrets Manager or similar

The goal is to reduce blast radius at every layer.

### Scenario. An EC2 instance in a private subnet suddenly becomes unreachable. No changes were deployed. What are the first things you check?

I would check:

1. Whether the instance is still running and passes both instance and system health checks.
2. The Security Group rules for the required inbound and outbound traffic.
3. The subnet NACL rules in both directions.
4. The route table for correct routes to NAT, VPN, or internal networks.
5. Whether I can reach it through Systems Manager Session Manager.
6. Whether the target application process is still listening on the expected port.

### Scenario. Your EKS node group is not scaling up during a traffic spike even though HPA triggered. How do you debug?

I first confirm that HPA actually increased the desired replica count with `kubectl describe hpa`. Then I check whether pods are stuck in `Pending`, which indicates lack of capacity. After that I inspect Cluster Autoscaler logs and verify whether the node group hit its maximum size, whether the pending pods can fit on any instance type, and whether taints, tolerations, or missing IAM permissions are blocking node scale-up.

---

## 3. Azure

### Q1. What is the difference between AKS and a self-managed Kubernetes cluster on Azure VMs?

AKS provides a managed control plane. Microsoft manages etcd, the API server, scheduler, controller manager, upgrades, and high availability. With a self-managed cluster, I would manage all of that myself. AKS significantly reduces operational overhead, which lets the team focus on application reliability rather than control plane maintenance.

### Q2. What is Azure Key Vault, and what types of secrets can it store?

Azure Key Vault is a managed service for secrets, certificates, and cryptographic keys. Secrets can store arbitrary key-value strings such as passwords or tokens. Certificates can be stored and rotated. Keys can be HSM-backed for cryptographic operations. We used it as a source of truth for production secrets and certificate material.

### Q3. How does Cert-Manager integrate with Azure Key Vault for TLS certificate management?

Cert-Manager handles certificate issuance and renewal inside Kubernetes. The certificate can then be synced to Azure Key Vault through the Secrets Store CSI Driver or a similar mechanism. The main value is automation. Cert-Manager renews the cert, and the refreshed material can be made available to workloads or external systems through Key Vault.

### Q4. Explain Azure RBAC vs Kubernetes RBAC. How do they interact in AKS?

Azure RBAC controls access to Azure resources, including the AKS cluster as an Azure resource. Kubernetes RBAC controls what an authenticated identity can do inside the cluster. In AKS with Azure AD integration, Azure identities can be mapped to Kubernetes roles and bindings, which centralizes identity management while still using Kubernetes-native authorization.

### Q5. How do you upgrade an AKS cluster with zero downtime across multiple node pools with stateful workloads?

I upgrade the control plane first, then upgrade node pools one at a time using surge settings so new nodes are added before old ones are drained. For stateful workloads, I ensure PodDisruptionBudgets are in place, verify storage behavior, and upgrade user pools carefully while monitoring pod rescheduling and readiness. Since AKS does not support downgrade, staging validation is critical.

### Scenario. On AKS, a pod cannot pull secrets from Azure Key Vault and workload identity is configured. Walk me through troubleshooting.

I would:

1. Describe the pod and look for CSI or mount-related errors.
2. Verify the Service Account has the correct workload identity annotation.
3. Check that the managed identity has the required Key Vault access.
4. Verify Key Vault firewall and network restrictions.
5. Inspect the `SecretProviderClass` for correct vault name, tenant ID, and object names.
6. Review the CSI driver logs in the cluster.

---

## 4. Bash / Shell Scripting

### Q1. What is the difference between `$@` and `$*` in a shell script?

Both represent all positional parameters. The important difference is when they are quoted. `"$@"` preserves each argument as a separate item, even if an argument contains spaces. `"$*"` combines everything into a single string. In production scripts I use `"$@"` because it is safer and avoids argument-splitting bugs.

### Q2. How do you handle errors in a Bash script? What does `set -euo pipefail` do?

I normally start scripts with `set -euo pipefail`.

- `-e` makes the script exit if a command fails.
- `-u` treats unset variables as errors.
- `pipefail` makes a pipeline fail if any command inside it fails.

This prevents silent failures and makes scripts safer in production.

### Q3. Write a Bash script that monitors disk usage across all mount points and sends an alert if any exceeds 80 percent.

```bash
#!/bin/bash
set -euo pipefail

THRESHOLD=80
ALERT_EMAIL="ops@company.com"

df -h --output=pcent,target | tail -n +2 | while read -r usage mount; do
  percent="${usage%%%}"
  if [[ "$percent" -gt "$THRESHOLD" ]]; then
    echo "ALERT: $mount is at ${percent}% usage" | \
      mail -s "Disk Alert: $mount at ${percent}%" "$ALERT_EMAIL"
  fi
done
```

In practice I would often send alerts to Slack or route them through monitoring rather than mail.

### Q4. How would you schedule a script to run every day at 2 AM and log its output?

I would use cron:

```cron
0 2 * * * /opt/scripts/my-script.sh >> /var/log/my-script.log 2>&1
```

That appends both standard output and error output to the log file. I would also pair it with log rotation.

### Q5. How do you handle signals in a long-running Bash script gracefully?

I use `trap` so the script can clean up before exiting.

```bash
cleanup() {
  echo "Caught signal, cleaning up"
  exit 1
}

trap cleanup SIGINT SIGTERM
```

This matters for long-running automation or containerized scripts where graceful shutdown is important.

### Scenario. You need to automate TLS certificate rotation across 20 servers. The new cert is in S3. Describe the logic for a safe rotation script with rollback.

The safe approach is:

1. Download the new certificate and key from S3 to a temporary location.
2. Validate the certificate before deployment.
3. Back up the current certificate and key on each server.
4. Copy the new certificate and key to the server.
5. Test the application or web server config before reloading.
6. Reload the service, not restart, if possible.
7. If validation or reload fails, restore the backup and reload the old configuration.

The important part is rollback and pre-validation. It should never blindly overwrite certs without testing.

---

## 5. Python

### Q1. What is the difference between a list and a tuple? When would you use each?

A list is mutable, so I can append, remove, or change elements. A tuple is immutable, so it is useful for fixed data that should not be modified. I use lists for collections that change over time and tuples for fixed structures or values that should remain constant.

### Q2. How do you handle exceptions in Python?

I use `try`, `except`, `else`, and `finally` blocks as needed. I prefer catching specific exceptions so the behavior is intentional. I use a broad exception only at the top level if I need to log and re-raise unexpected failures.

### Q3. How would you use `boto3` to list all EC2 instances across all regions and find any without a Team tag?

The main steps are:

1. Use `describe_regions` to get the list of AWS regions.
2. Create an EC2 client per region.
3. Use a paginator for `describe_instances`.
4. Build a tag dictionary for each instance.
5. Report instances where the `Team` tag is missing.

Using paginators is important because a single API call will not return everything in larger accounts.

### Q4. Explain Python's `subprocess` module. When would you use it over `os.system()`?

`subprocess` gives better control. I can capture output, inspect exit codes, set timeouts, pass arguments safely, and avoid shell injection by passing a list of arguments. `os.system()` is much more limited and is not suitable for serious automation.

### Q5. How do you write an async Python script to poll multiple Kubernetes pod statuses concurrently?

I would use `asyncio` along with an async Kubernetes client such as `kubernetes_asyncio`. Each pod status fetch becomes an awaitable task, and `asyncio.gather()` can be used to run all of them concurrently. This reduces total wait time when polling a large number of pods.

### Scenario. Write a Python script that checks all Kafka consumer groups for lag greater than 10,000 messages and posts an alert to Slack.

The overall flow would be:

1. Connect to Kafka Admin APIs and list consumer groups.
2. Get committed offsets for each group.
3. Fetch end offsets for each topic-partition.
4. Calculate lag as end offset minus committed offset.
5. If lag exceeds 10,000, accumulate alert data.
6. Send a formatted alert to Slack through a webhook.

The key point is that the logic is simple, but productionizing it means handling auth, errors, and large numbers of groups safely.

---

## 6. CI/CD (Jenkins)

### Q1. What is the difference between a Declarative and a Scripted Jenkins pipeline?

Declarative pipelines use a structured DSL with a predefined format. They are easier to read, validate, and maintain. Scripted pipelines use Groovy and are more flexible, but also more complex and harder to govern at scale. I prefer Declarative for standard CI/CD workflows and use Scripted only when I need advanced control flow.

### Q2. What is a Jenkinsfile and where should it live?

A Jenkinsfile defines the pipeline as code. It should live in the application repository so the build and deployment process is versioned with the code itself. That makes changes auditable and easier to review.

### Q3. How do you implement parallel stages in a Jenkins pipeline? What are the trade-offs?

Parallel stages are defined under a `parallel` block so multiple tasks such as tests, linting, and scans can run at the same time. The main benefit is reduced pipeline duration. The trade-offs are higher resource usage on Jenkins agents and slightly more complicated debugging when one branch fails.

### Q4. How do you manage secrets in Jenkins pipelines securely?

I store secrets in Jenkins Credentials or an external secret system such as Azure Key Vault or AWS Secrets Manager. Pipelines access them through bindings such as `withCredentials`, which keeps secrets out of source control and masks them in logs.

### Q5. How do you implement a blue-green deployment strategy in a Jenkins pipeline for Kubernetes?

I deploy the new version to the inactive environment, such as green if blue is live. Then I wait for rollout completion, run smoke tests, and only after validation switch the service selector or traffic routing to the new environment. I keep the old environment available temporarily for fast rollback.

### Q6. How would you structure Jenkins shared libraries for 250+ microservices to avoid duplication?

I centralize reusable pipeline logic in a shared library repository. Common steps such as build, scan, deploy, and notify become reusable functions. Individual service Jenkinsfiles stay thin and only pass service-specific parameters. I also version the shared library so changes can be rolled out safely.

### Scenario. Your Jenkins pipeline takes 60 minutes to complete. You need to bring it under 15 minutes. What steps do you take?

I would first profile which stages consume the most time. Then I would parallelize independent stages, cache dependencies, optimize Docker layer caching, avoid unnecessary work on every branch, and distribute heavy tasks across more agents. In many cases, test parallelization and dependency caching alone make a major difference.

### Scenario. A pipeline passes in CI but fails in production deployment. Rollback is needed. How is this handled automatically?

I structure the deployment stage so that rollout status and smoke tests must pass. If any of those fail, the pipeline automatically triggers a rollback using Helm or the deployment tool in use. Notifications are sent immediately so the team knows the rollback happened and can investigate the root cause.

---

## 7. Kubernetes

### Q1. What is the difference between a Deployment and a StatefulSet?

A Deployment is for stateless applications where replicas are interchangeable. A StatefulSet is for stateful workloads that need stable identity, ordered startup and shutdown, and stable storage per replica. I use StatefulSets for Kafka, ClickHouse, and similar systems.

### Q2. What is a PersistentVolumeClaim? How does it bind to a PersistentVolume?

A PVC is a request for storage from a workload. It defines size, access mode, and optionally a StorageClass. Kubernetes binds it to a matching PersistentVolume or dynamically provisions one through the storage provider. Once bound, that PVC is associated with that volume.

### Q3. Explain the role of `kube-proxy` in a Kubernetes cluster.

`kube-proxy` implements the Service networking abstraction on each node. It watches Services and Endpoints and programs iptables or IPVS rules so traffic to a ClusterIP gets forwarded to the appropriate pod backends.

### Q4. How does HPA work? What metrics can it scale on? What are its limitations?

HPA adjusts replica count based on observed metrics. By default it uses CPU and memory through Metrics Server, but with custom or external metrics it can scale on more meaningful signals. Its limitations include delayed reaction time, dependency on properly set resource requests, and the fact that it cannot solve node capacity issues by itself.

### Q5. Explain Kubernetes RBAC.

Roles and RoleBindings are namespace-scoped. ClusterRoles and ClusterRoleBindings are cluster-wide. The idea is to define what actions are allowed on which resources and bind those permissions to users, groups, or Service Accounts. I keep permissions narrow and avoid wildcard privileges.

### Q6. What are init containers? Give a real use case.

Init containers run before the main application container and must complete successfully before the pod becomes ready. Common use cases include waiting for a database to become available, downloading startup assets, or running migrations.

### Q7. How do Kubernetes network policies work at the CNI level? How would you implement zero-trust pod networking?

Network policies are enforced by the CNI plugin, such as Calico or Cilium. I would start with a default deny policy for both ingress and egress, then explicitly allow only the traffic required between services, DNS, and monitoring. That creates a zero-trust posture inside the cluster.

### Q8. What is a PodDisruptionBudget? How does it interact with Cluster Autoscaler and rolling updates?

A PDB defines how many replicas must remain available during voluntary disruptions such as node drains or rolling upgrades. Cluster Autoscaler respects PDBs when scaling down nodes. If the PDB is too strict, it can prevent node scale-down or make upgrades slower.

### Q9. How do you debug a node that shows `NotReady`?

I describe the node to inspect its conditions, then check kubelet logs, container runtime health, disk usage, memory pressure, and network connectivity to the API server. Most node issues are caused by disk, kubelet, or networking problems.

### Scenario. A pod in production enters CrashLoopBackOff. You cannot reproduce it locally. Walk me through your live debugging steps.

I start with `kubectl describe pod` and `kubectl logs --previous` to inspect exit codes and prior container logs. Then I check resource pressure, environment variables, mounted config, secret values, and recent config changes. If it is OOMKilled, I focus on memory limits and usage. If it is an app startup failure, I compare the pod's runtime environment to a working environment.

### Scenario. You right-sized 250 workloads and improved cluster efficiency by 30 percent. What process did you follow to avoid throttling and OOM issues?

I based the changes on real usage data collected over time, not guesses. I grouped workloads by behavior, adjusted requests and limits incrementally in waves, watched CPU throttling and OOM metrics closely after each wave, and revalidated HPA behavior after each resource change. That reduces cost without introducing instability.

---

## 8. Helm

### Q1. What is the difference between `helm install` and `helm upgrade --install`?

`helm install` only works if the release does not already exist. `helm upgrade --install` is idempotent because it installs the release if missing or upgrades it if present. In CI/CD this is usually the better choice.

### Q2. What is a Helm hook? Give an example use case.

A Helm hook is a Kubernetes resource that executes at a specific point in the release lifecycle, such as pre-install, post-install, or pre-upgrade. A common use case is running a database migration job before deploying a new application version.

### Q3. How do you manage environment-specific overrides in Helm values?

I keep a base `values.yaml` file and layer environment-specific files such as `values-staging.yaml` or `values-production.yaml` on top during deployment. Dynamic values such as image tags are passed separately with `--set` or generated values files.

### Q4. How do you test a Helm chart before deploying it to production?

I use `helm lint`, `helm template`, and dry-run validation. After that I deploy to a non-production environment and run smoke tests. For critical applications I also use a diff step to inspect exactly what will change.

### Q5. How do you implement a library Helm chart shared across 250+ microservices?

I use a Helm library chart to centralize common templates such as Deployments, Services, HPAs, and labels. Individual service charts consume that library and provide only values and service-specific overrides. This keeps chart logic standardized across all services.

### Scenario. A `helm upgrade` is mid-rollout and you notice the new pods are crashing. ArgoCD auto-sync is enabled. What happens and how do you intervene?

ArgoCD will continue to enforce the desired state from Git, which may interfere with manual rollback attempts. I would temporarily disable or pause auto-sync, perform a rollback to the previous healthy release, verify recovery, fix the issue properly, and only then re-enable sync.

---

## 9. Docker

### Q1. What is the difference between `CMD` and `ENTRYPOINT` in a Dockerfile?

`ENTRYPOINT` defines the main executable of the container. `CMD` provides default arguments or a default command if no entrypoint is set. `CMD` is easily overridden at runtime, whereas `ENTRYPOINT` is intended to keep the primary executable fixed.

### Q2. What is a multi-stage Docker build and why would you use it?

A multi-stage build separates the build environment from the final runtime image. That allows me to compile or build in a heavy image and copy only the final artifact into a small runtime image. The result is a smaller, more secure image with fewer dependencies.

### Q3. How do Docker bridge, host, and overlay networks differ?

Bridge networking is the default local container network on a host. Host networking shares the host's network stack directly with the container. Overlay networking spans multiple hosts and is used for distributed container networking in orchestrated systems such as Swarm. In Kubernetes, CNI handles networking rather than standard Docker networking modes.

### Q4. How do you reduce Docker image size? Name at least four techniques.

I reduce image size by:

- Using multi-stage builds
- Choosing minimal base images
- Cleaning package caches in the same layer where packages are installed
- Using a `.dockerignore` file to exclude unnecessary files
- Ordering layers to maximize caching efficiency

### Q5. What are the security risks of running containers as root? How do you mitigate them?

Running as root increases the impact of a container escape or misconfiguration. I mitigate this by running as a non-root user, dropping Linux capabilities, using a read-only root filesystem where possible, and setting pod security controls such as `allowPrivilegeEscalation: false`.

### Scenario. A containerized service is consuming four times more memory than expected in production but not in dev. How do you investigate and fix this?

I confirm actual memory usage first, then compare production and dev differences such as traffic levels, data volume, feature flags, and connection pool sizes. I look for memory leaks, cache growth, unbounded buffers, or workload-specific behavior. The fix depends on the root cause, but the path is always measure, compare, isolate, and then tune or patch.

---

## 10. Terraform

### Q1. What is Terraform state and why is it important?

Terraform state maps the code to the real infrastructure. It stores resource identifiers, attributes, and dependency information. Terraform uses state to know what already exists and what must change. In team environments, remote state with locking is essential to avoid collisions and corruption.

### Q2. What is the difference between `terraform plan` and `terraform apply`?

`terraform plan` calculates and shows the changes Terraform wants to make. `terraform apply` executes those changes. I prefer generating a plan file and applying exactly that plan so reviewed changes match what is deployed.

### Q3. How do you manage Terraform state for multiple teams working on the same infrastructure?

I separate state by environment and ownership boundary. Shared platform components have their own state, and each team has independent state for team-owned infrastructure. I use remote backends with locking and expose shared outputs cleanly instead of merging unrelated ownership into one huge state file.

### Q4. What is a Terraform module and what makes a good reusable module?

A module is a reusable package of Terraform configuration. A good module has a single responsibility, well-defined variables, useful outputs, sensible defaults, and good documentation. It should also be versioned so consumers can adopt updates safely.

### Q5. How do you handle Terraform state drift detection and remediation in production?

I detect drift by running scheduled plans in CI/CD or a platform pipeline. If drift is found, I first understand whether it was a legitimate emergency change or unauthorized manual change. Then I either update code to match the new desired state or apply Terraform to bring infrastructure back in line.

### Q6. How do you implement DRY Terraform across AWS and Azure with the same logical structure?

I use modules and consistent abstractions. The cloud-specific implementation lives inside provider-specific modules, but the higher-level composition pattern remains the same. That keeps the layout familiar across clouds without forcing a fake one-size-fits-all resource model.

### Scenario. `terraform plan` shows that three production resources will be destroyed unexpectedly. What do you do before running `terraform apply`?

I stop immediately and investigate. I inspect the exact resources, the change causing replacement or destroy, provider version differences, recent code changes, and whether state drift or import issues are involved. I do not apply until I can clearly explain why Terraform wants to destroy those resources.

### Scenario. A new team wants a fresh AWS environment in two hours using your Terraform modules. Walk me through the handoff process.

I would instantiate the standard environment stack using the approved modules, apply team-specific variables, review the plan, provision the infrastructure, and then hand over the resulting outputs and onboarding documentation. The value of modules is that the setup becomes consistent, fast, and repeatable.

---

## 11. Ansible

### Q1. What is the difference between an Ansible playbook and a role?

A playbook is the top-level orchestration that defines what should run on which hosts. A role is a reusable package of tasks, handlers, templates, files, variables, and defaults. Roles help structure automation and reduce duplication.

### Q2. What is idempotency and why does it matter in Ansible?

Idempotency means running the same automation multiple times produces the same desired result without causing unintended changes. This matters because configuration management should be safe to re-run at any time.

### Q3. How do you use Ansible Vault to manage secrets?

I encrypt sensitive variables or files using Ansible Vault and decrypt them only at execution time. That allows the repository to remain version-controlled without exposing plaintext secrets.

### Q4. What is the difference between handlers and regular tasks?

Regular tasks run in sequence every time. Handlers run only when notified by a task and typically execute at the end of a play. I use handlers for actions like restarting a service only when its configuration has changed.

### Q5. How would you use Ansible with a dynamic inventory from AWS EC2?

I would use the AWS EC2 dynamic inventory plugin so instances are discovered through tags, regions, and filters. That removes the need to manually maintain static host lists and fits cloud environments much better.

### Scenario. An Ansible playbook has run on 18 of 20 servers and failed on 2 mid-playbook. What state are your servers in and how do you recover safely?

The successful servers may already be fully configured, while the failed servers may be partially changed. I inspect the failure cause, determine which tasks already ran, fix the issue, and re-run safely against the failed hosts. For critical workflows I use blocks and rescue logic to handle partial failure more cleanly.

---

## 12. Prometheus

### Q1. What are the four metric types in Prometheus? When do you use each?

The four metric types are Counter, Gauge, Histogram, and Summary.

- Counter is for values that only increase, such as total requests.
- Gauge is for values that go up and down, such as memory usage.
- Histogram is for distributions, such as request latency buckets.
- Summary also tracks distributions, but client-side, and is less flexible for cross-instance aggregation.

### Q2. What is a scrape interval and how does it affect alerting latency?

The scrape interval defines how often Prometheus collects metrics from a target. Alerting latency depends on scrape interval, rule evaluation interval, and any `for` duration configured on the alert. Shorter intervals improve detection speed but increase load.

### Q3. Write a PromQL query to find the 5-minute rate of HTTP 5xx errors as a percentage of total requests.

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100
```

### Q4. What is a recording rule and when would you create one?

A recording rule precomputes a PromQL query and stores the result as a new time series. I create recording rules for expensive queries, commonly reused dashboard queries, or alert expressions that should evaluate faster and more consistently.

### Q5. How do you implement SLO burn-rate alerting for a 99.9 percent availability target?

I use a multi-window, multi-burn-rate approach. A fast-burn alert catches sudden severe failures using short windows, and a slow-burn alert catches sustained degradation over longer windows. This balances sensitivity with noise reduction.

### Scenario. Your team is drowning in alert noise with 200+ firing alerts, most of them flapping. What is your process to reduce alert fatigue?

I start by identifying which alerts fire most often and whether they have ever corresponded to a real incident. Then I tune thresholds and `for` durations, add inhibition rules, remove unactionable alerts, and make sure each alert has a clear owner and action. The goal is fewer alerts, but higher signal quality.

---

## 13. Grafana

### Q1. What is a Grafana data source? Name three you have used.

A data source is the backend Grafana queries for dashboards and alerts. Common examples I have used include Prometheus for metrics, Elasticsearch or Kibana-backed data through Elasticsearch for logs, and ClickHouse for analytical dashboards.

### Q2. What is the difference between a Grafana alert and a Prometheus alert rule?

Prometheus alert rules live with metrics and route through Alertmanager, which is strong for routing, silencing, and deduplication. Grafana alerts can work across multiple data sources and are useful for broader visualization-driven alerting. For core production monitoring, I prefer Prometheus and Alertmanager.

### Q3. How do you use template variables in Grafana dashboards to make them reusable across services?

I define variables such as namespace, service, cluster, or instance and use them in panel queries. That allows one dashboard to be reused by many services without copying or duplicating dashboard definitions.

### Q4. How do you provision Grafana dashboards and alert rules as code? What are the benefits?

I provision them using JSON or YAML files stored in Git and loaded into Grafana through config or Kubernetes ConfigMaps. The main benefits are version control, reproducibility, reviewability, and easy promotion across environments.

### Scenario. A Grafana dashboard shows p99 latency spiking to 2 seconds for one microservice at 3 AM. You are on call. Walk me through your response.

I acknowledge the alert, assess impact through SLO dashboards, isolate whether the issue affects one endpoint or the whole service, correlate with traffic and infrastructure metrics, and inspect logs and recent deployments. Then I mitigate by rolling back, scaling, or removing the bottleneck depending on what the evidence shows.

---

## 14. ELK / EFK Stack

### Q1. What are the components of the EFK stack and the role of each?

Fluent Bit or Fluentd collects and forwards logs. Elasticsearch stores and indexes the logs. Kibana provides search and visualization. Together they provide centralized logging and analysis.

### Q2. What is the difference between Logstash and Fluentd or Fluent Bit?

Logstash is powerful but heavier and more resource-intensive. Fluent Bit is lightweight and well-suited for node-level collection in Kubernetes. Fluentd is more feature-rich than Fluent Bit and is used when more processing is needed. I typically use Fluent Bit for collection and keep the pipeline as lightweight as possible.

### Q3. How do you implement index lifecycle management in Elasticsearch to control disk usage?

I define ILM policies that move indices through hot, warm, cold, and delete phases based on age or size. This keeps disk usage predictable and ensures old data is deleted or moved appropriately.

### Q4. How do you parse structured JSON logs vs unstructured logs in Fluentd?

Structured JSON logs are straightforward because the parser can extract fields directly. Unstructured logs require regex or custom parsing rules. I strongly prefer structured logging because it reduces parsing complexity and improves downstream query quality.

### Q5. Your Elasticsearch cluster is at 85 percent disk capacity and indexing is throttling. What immediate and long-term actions do you take?

Immediately I would free space by applying or accelerating lifecycle policies, deleting stale or low-value indices if appropriate, and reducing non-critical replicas if necessary. Long-term I would refine retention, reduce unnecessary log volume, tune shard strategy, and expand storage if growth is legitimate.

### Scenario. A service is not appearing in Kibana logs despite being deployed 30 minutes ago. How do you trace the log pipeline?

I start at the application and verify logs are being written to stdout or stderr. Then I check the collector on that node, inspect Fluent Bit or Fluentd logs, verify forwarding to Elasticsearch, check the index itself, and finally validate Kibana index patterns and time filters. The goal is to locate the exact handoff where logs stop flowing.

---

## 15. Networking / Nginx / DNS

### Q1. What is the difference between an A record and a CNAME record in DNS?

An A record maps a hostname directly to an IP address. A CNAME maps one hostname to another hostname. A CNAME is useful for aliasing, while an A record is the direct final mapping.

### Q2. What does Nginx do as a reverse proxy vs a load balancer?

As a reverse proxy, Nginx accepts client traffic and forwards it to backend services. As a load balancer, it distributes traffic across multiple backends according to the chosen strategy. In practice, it often serves both roles together.

### Q3. How do you configure Nginx for SSL termination? Where should SSL be terminated in Kubernetes?

Nginx is configured with the certificate and private key, then it handles HTTPS from clients and forwards traffic to backend services. In Kubernetes, SSL is commonly terminated at the Ingress controller unless end-to-end encryption is required.

### Q4. What is DNS TTL and how does a low vs high TTL affect failover time?

TTL controls how long DNS resolvers cache a record. Low TTL means faster failover but more frequent DNS queries. High TTL reduces query load but slows failover because clients keep old answers longer.

### Q5. How do you implement Nginx rate limiting to protect a downstream service from abuse?

I define a rate limiting zone and apply it to the target location or server block. I tune the request rate, burst handling, and response code. The purpose is to smooth abusive or accidental traffic spikes before they overwhelm the backend.

### Scenario. Users are reporting intermittent 502 errors on a service. The backend pods are healthy. Nginx access logs show upstream timeouts. How do you diagnose and fix this?

I inspect Nginx timeout settings, confirm backend readiness and response time, test connectivity from the ingress layer to the backend, inspect Service endpoints, review network policies, and correlate with backend latency metrics. Healthy pods do not automatically mean healthy response paths, so I focus on real request timing rather than just pod status.

---

## 16. Kafka

### Q1. What is the role of a Kafka topic, partition, offset, and consumer group?

A topic is the logical stream of messages. Partitions split the topic for scalability and parallelism. Offsets are the ordered positions of messages within a partition. A consumer group is a set of consumers that share the work of reading partitions.

### Q2. What happens when a Kafka consumer dies mid-consumption?

The consumer stops heartbeating, the group coordinator detects that after the session timeout, and a rebalance occurs. The partitions owned by the failed consumer are reassigned to others. Consumption resumes from the last committed offset, so reprocessing may happen if messages were read but not committed.

### Q3. How does Kafka replication work? What is ISR?

Each partition has a leader and one or more followers. Producers write to the leader, and followers replicate from it. ISR means In-Sync Replicas, which are the replicas that are sufficiently caught up with the leader. Only in-sync replicas are considered safe candidates for leader election.

### Q4. How do you tune Kafka producer throughput vs latency?

For higher throughput I increase batch size, increase linger slightly, and enable compression. For lower latency I reduce or eliminate linger, keep smaller batches, and accept the trade-off of lower efficiency. The right configuration depends on whether the workload values speed of acknowledgment or total throughput more.

### Q5. Your Kafka cluster is receiving 2 million messages per second from MQTT. How do you design partition count, replication factor, and retention for this scale?

I start from expected throughput per partition and size partition count with headroom for consumers and future growth. I use a replication factor of 3 for durability. Retention depends on downstream recovery and replay needs, but I avoid storing more hot data than necessary and use tiering or downstream archival where appropriate.

### Scenario. Consumer lag on a critical topic spikes from 100 to 500,000 messages in 10 minutes. What do you do immediately and what is your root cause investigation?

Immediately I verify consumer health, processing rate, rebalance events, and downstream dependencies. If needed, I scale consumers or reduce the bottleneck. For root cause, I determine whether the issue was a traffic spike, consumer slowdown, a downstream dependency such as a database, or partition imbalance.

---

## 17. ArgoCD / GitOps

### Q1. What is GitOps? How does ArgoCD implement it?

GitOps uses Git as the source of truth for infrastructure and application state. ArgoCD continuously compares the desired state stored in Git with the live state in Kubernetes and reconciles differences. That gives auditable, repeatable deployments with an easy rollback path.

### Q2. What is the difference between ArgoCD Sync and Refresh?

Refresh updates ArgoCD's view of Git and cluster state. Sync actually applies the desired state from Git to the cluster. Refresh is observational, while Sync is corrective.

### Q3. What is the App of Apps pattern in ArgoCD? When would you use it?

The App of Apps pattern means one parent ArgoCD application manages a set of child ArgoCD applications. I use it for cluster bootstrapping and multi-team environments because it makes large-scale GitOps management easier and more structured.

### Q4. How do you manage secrets in a GitOps workflow where the Git repo must not contain plaintext secrets?

I use External Secrets Operator, Sealed Secrets, or a Vault integration. My preference is external secret management so Git contains only references or encrypted material, never plaintext credentials.

### Q5. How do you implement progressive delivery using ArgoCD Rollouts?

I define rollout strategies such as canary or blue-green in Rollout resources, integrate them with ingress or service mesh for traffic splitting, and use automated analysis from metrics to decide whether to continue or abort the rollout.

### Scenario. An ArgoCD application is stuck in `Progressing` state for 20 minutes. The Git repo is correct. How do you debug and resolve it?

I inspect the application status, look at the resources that are not healthy or not synced, review Kubernetes events, and then narrow down whether the issue is a failed job, unhealthy deployment, storage problem, or policy rejection. The resolution depends on the exact resource that is blocking progress.

---

## 18. Databases

### Q1. What is the difference between a primary and replica in PostgreSQL?

The primary handles writes. Replicas receive WAL changes from the primary and can serve read traffic. Replicas improve read scalability and high availability, but replication lag must be monitored because stale reads are possible.

### Q2. What is Redis eviction policy and when would you set `allkeys-lru`?

Redis eviction policy defines how keys are removed when memory is full. `allkeys-lru` evicts the least recently used key from the entire keyspace. I use it when Redis is acting purely as a cache and any key can be evicted safely.

### Q3. How does ClickHouse achieve high-performance analytical queries? What makes it different from PostgreSQL for analytics?

ClickHouse is column-oriented, so analytical queries read only the required columns rather than full rows. It also uses strong compression, vectorized execution, and efficient storage engines such as MergeTree. PostgreSQL is row-oriented and excellent for transactional workloads, but not optimized for the same style of high-volume analytics.

### Q4. How do you troubleshoot slow queries in PostgreSQL? What tools and views do you use?

I use `pg_stat_statements` to identify expensive queries, `EXPLAIN ANALYZE` to inspect query plans, `pg_stat_activity` for live sessions, and lock-related views when contention is suspected. Then I improve indexes, query design, and connection management based on what the evidence shows.

### Q5. How do you tune Redis or Dragonfly connection pooling for a high-throughput microservice environment?

I size connection pools based on concurrency, instance count, and backend capacity. I set upper bounds, aggressive but safe timeouts, and monitor connection counts and latency. The goal is to avoid connection storms while keeping latency low.

### Scenario. ClickHouse p99 read latency spikes to 4 seconds during peak ingest. Writes and reads share the same cluster. What is your diagnosis and remediation plan?

I suspect read and write contention, especially background merges competing for disk I/O. I would inspect active merges and query activity, validate whether specific queries are heavy, and then separate read and write roles if needed, tune merge behavior, cache frequent reads, or scale the cluster appropriately.

---

## 19. SRE Practices

### Q1. What is the difference between SLA, SLO, and SLI? Give a real example.

An SLI is the actual measured indicator, such as request success rate. An SLO is the internal reliability target, such as 99.9 percent success over a month. An SLA is the external agreement with customers and may include penalties. I keep the SLO stricter than the SLA so the team has a safety buffer.

### Q2. What is an error budget and how is it calculated?

An error budget is the allowed amount of unreliability under an SLO. For a 99.9 percent monthly SLO, the budget is 0.1 percent of total monthly time. That works out to roughly 43 minutes of allowed downtime per 30-day month.

### Q3. How do you design an SLO for an MQTT broker ingesting real-time vehicle telemetry?

I choose SLIs that match user and system expectations, such as connection success rate and publish-to-downstream delivery latency. Then I define objectives for those metrics, for example 99.9 percent connection success and p99 delivery latency under a defined threshold.

### Q4. What makes a good postmortem? What sections should it always include?

A good postmortem is blameless, factual, and action-oriented. It should include a timeline, impact summary, root cause, contributing factors, what went well, and specific follow-up action items with owners and due dates.

### Q5. Your error budget for a service is 40 percent burned in the first week of the month. As the SRE, what actions do you take?

I investigate the source of the burn immediately, communicate risk clearly, and usually slow or freeze non-critical changes until the reliability issue is addressed. Error budget should influence release velocity. It is not just a reporting metric.

### Q6. How do you balance feature velocity with reliability when engineering teams want to ship fast?

I use error budgets and SLOs as the decision framework. When reliability is healthy, teams can move quickly. When budget burn is too high, the priority shifts to stability. That keeps the conversation objective rather than opinion-based.

### Scenario. It is 2 AM. PagerDuty fires a P0: MQTT broker cluster, all consumers offline, 2 million vehicles not reporting telemetry. Walk me through your incident response.

I acknowledge the incident, declare severity, start communication immediately, verify impact, and begin triage by checking broker pods, nodes, recent changes, and downstream dependencies. I mitigate first, such as restoring service, scaling, or rolling back. Once the platform is stable, I coordinate deeper investigation and follow with a blameless postmortem.

---

## 20. Auto Scaling

### Q1. What is the difference between HPA, VPA, and Cluster Autoscaler?

HPA scales the number of pods. VPA adjusts pod resource requests and limits. Cluster Autoscaler scales the number of cluster nodes. They solve different levels of capacity management and should be used deliberately so they do not interfere with one another.

### Q2. When would you use KEDA over HPA?

I use KEDA when scaling should be driven by external event sources such as Kafka lag, queue depth, cron schedules, or similar signals. HPA is strong for resource-based scaling, but KEDA is better when the real work signal exists outside CPU or memory usage.

### Q3. How do you configure KEDA to scale a consumer deployment based on Kafka consumer lag?

I define a `ScaledObject` referencing the deployment, specify Kafka connection metadata, the consumer group, the topic, and thresholds for lag. KEDA then calculates desired replicas from observed lag and updates the workload accordingly.

### Q4. What are the risks of HPA and Cluster Autoscaler fighting each other?

The main risk is flapping. HPA can scale pods down, which makes nodes look empty, and Cluster Autoscaler may remove those nodes. Then HPA or load growth can require new nodes again. This is mitigated with stabilization windows, realistic minimums, and sensible autoscaler timing.

### Q5. How did you achieve around 40 percent cost reduction using KEDA, PerfectScale, and node-pool consolidation without p99 latency regression? Walk me through the approach.

I started with real usage analysis to right-size workloads. Then I replaced wasteful always-on scaling patterns with event-driven scaling where appropriate. Finally, I consolidated node pools to improve scheduling efficiency and reduce fragmentation. Each stage was validated against latency and reliability metrics before moving further.

### Scenario. A sudden 10x traffic spike hits at 9 AM. HPA triggers, but new pods take 4 minutes to become ready. During those 4 minutes, the existing pods are overwhelmed. How do you architect the system to handle this better?

I would reduce startup time, introduce predictive or scheduled pre-scaling if the pattern is known, keep some warm capacity available, and where appropriate place a queue or buffering layer in front of the service. The core principle is that pure reactive scaling is not enough when startup time is longer than the spike tolerance window.

---

## Final Note

These answers are written in the style of a confident, experienced interviewee who explains clearly, ties concepts back to real production behavior, and focuses on practical reasoning instead of textbook definitions alone.