# Operations

How the platform is deployed, upgraded, checked and recovered.

## Delivery

Rancher Fleet watches Git repositories and applies the Kubernetes resources they contain. This makes the intended deployed state reviewable in Git, and a rollback is normally a Git revert. I use `kubectl` and Helm directly only for an initial operator installation or an urgent recovery action. Any lasting manual change is then added to the Git-managed configuration.

Pulumi creates virtual machines; Fleet deploys Kubernetes workloads. Keeping these responsibilities separate prevents two tools from attempting to manage the same resource.

## Change management

Changes follow a defined order. Each step has a way to roll back if validation fails.

1. Read the release notes for the target RKE2 and Cilium versions
2. Upgrade one control-plane node, then check etcd health and the Kubernetes API address.
3. Upgrade the remaining control-plane nodes, then worker nodes, draining one node at a time.
4. Upgrade Cilium, Longhorn and Traefik as their charts require
5. Check `cilium status` and make a request to a LAN `LoadBalancer` address.
6. Run basic user checks: Keycloak sign-in, Joplin note sync and Homepage load.

Flatcar updates use a second disk partition, so the node can return to the previous version if an update fails. Helm releases can return to their prior revision. Fleet applies a rollback when the relevant Git commit is reverted.

RKE2's system upgrade controller performs Kubernetes node upgrades. The upgrade plan is a Kubernetes resource instead of a handwritten set of commands. Related changes are grouped into one maintenance window so the number of concurrent changes stays small and faults are easier to identify.

## State and recovery

The platform has three kinds of state. Each needs a different recovery method.

**Infrastructure and workload definition.** Pulumi contains virtual machine specifications. Git repositories contain workload manifests, and Fleet applies them. To replace a Kubernetes node, Pulumi creates a new virtual machine instead of repairing the old one by hand.

**Kubernetes cluster state.** RKE2 creates scheduled etcd snapshots on control-plane nodes. These snapshots cover damage to the Kubernetes database when the underlying host remains available.

**Application data.** Longhorn replicates volumes across nodes. CloudNativePG runs each PostgreSQL database as a primary and replica pair. Replication protects against an individual node failure. It does not protect against accidental deletion, data corruption or loss of the host carrying most of the cluster. The planned off-cluster backup destination addresses these cases.

Vaultwarden stores platform credentials. Secrets are never stored in the Git repositories that hold deployment configuration.

## Failure behaviour

I run these exercises deliberately so the expected result is known before an unplanned failure.

| Exercise | Observed behaviour |
| --- | --- |
| Drain the worker that advertises a Cilium service address | Another worker takes the lease, sends an Address Resolution Protocol (ARP) update and continues serving the address. |
| Stop one control-plane virtual machine | kube-vip moves the Kubernetes API address; etcd retains a majority of members. |
| Drain a Longhorn node | Longhorn rebuilds replicas on another node while the volume remains attached. |
| Trigger a CloudNativePG switchover | CloudNativePG promotes the replica and the application reconnects to the new primary. |
| Disconnect the internet connection | LAN services remain available; public tunnelled hostnames recover when the connection returns. |
| Apply a restrictive NetworkPolicy | Cilium shows that it blocked the connection, rather than leaving an unexplained timeout. |

I validate loss of the primary Proxmox host through the infrastructure rebuild procedure rather than by switching off the active host. Turning off the primary host would interrupt the services it currently runs. That rebuild confirms Pulumi can recreate virtual machines and Fleet can reapply declared workloads. It is not yet a full disaster-recovery test for application data, because the off-cluster backup destination below does not exist yet.

## What I am building next

The next work improves backup, monitoring and incident investigation. The sequence matters because each stage provides information or storage needed by the next.

**Off-cluster backups.** A network-attached storage (NAS) device, separate from both Proxmox hosts, will store Longhorn backups and CloudNativePG object storage. PostgreSQL write-ahead log (WAL) archiving to that destination will enable point-in-time recovery. A backup stored on the same host as the original data would not protect against a host failure, so current replication is described as replication rather than backup recovery.

**Metrics and dashboards.** Prometheus will collect cluster and application metrics; Grafana will display them. Blackbox probes will check user-facing endpoints from outside Kubernetes. An external check is necessary because a monitoring system inside the cluster cannot report that the cluster itself is unreachable.

**Log aggregation.** Centralised application and platform logs will allow an investigation to start with a search across services rather than a manual check of individual pods.

**Service objectives.** Once probe data has accumulated, each service can receive an availability target based on measured behaviour. Publishing targets before collecting that data would create numbers that cannot be verified.

## After an incident

Each incident receives a short record: the symptom, cause, service restoration step and follow-up action. Repeated manual work is a candidate for automation because it consumes time that could otherwise improve the platform.
