# Platform

How the cluster is built, what each main component does, and why it was selected.

## Provisioning

The Kubernetes nodes are virtual machines in Proxmox. They are created in four steps:

1. Pulumi, written in Python, creates the control-plane and worker virtual machines through the Proxmox application programming interface (API).
2. Ignition configures Flatcar Linux at first boot with a hostname, static network address, SSH keys and RKE2.
3. The first control-plane node initialises the cluster, and the remaining nodes join.
4. kube-vip provides the Kubernetes API address, and Cilium is installed in place of kube-proxy for Service networking.

The Pulumi project contains the virtual machine configuration, the Proxmox provider, Ignition generation and `pulumi up`. The number of control-plane and worker nodes is configuration, not duplicated code. The current cluster has three control-plane nodes and two workers. Rebuilding a node is therefore a documented command sequence, not manual reconstruction.

```text
pulumi-homelab/
  __main__.py      # create the requested control-plane and worker nodes
  config.py        # sizing, image channel, address allocation
  proxmox.py       # VM creation and disk handling
  ignition.py      # Flatcar and RKE2 configuration
```

## Why these components

The table gives the short reason for each choice. The sections below explain the choices that have the greatest operational effect.

| Component | Chosen over | Deciding factor |
| --- | --- | --- |
| Proxmox VE | Bare metal, ESXi | Provides a KVM virtual machine API, snapshots and ZFS storage. |
| Flatcar Linux | Ubuntu, Talos | Immutable operating system with atomic updates and rollback. |
| RKE2 | kubeadm, k3s | Packages control-plane components into one distribution release, with a controlled node upgrade sequence. |
| Pulumi | Terraform | Python is useful for generating Ignition configuration and creating a configurable number of nodes. |
| Cilium | Calico plus MetalLB | Combines pod networking, service addresses, network policy and traffic inspection. |
| kube-vip | HAProxy and keepalived | Provides one Kubernetes API address without additional virtual machines. |
| Traefik | ingress-nginx | Ships with RKE2, reducing the number of independently upgraded components. |
| Longhorn | Rook and Ceph | Provides replicated storage with a lower resource requirement than Ceph. |
| CloudNativePG | PostgreSQL StatefulSets with cron jobs | Manages PostgreSQL replication, promotion and backup features. |
| Rancher Fleet | Argo CD | Applies Git-managed configuration through Rancher, which is already in use. |
| Cloudflare Tunnel | Port forwarding and dynamic DNS | Publishes applications without opening inbound firewall ports. |

### Flatcar Linux

Flatcar reduces configuration drift, where nodes gradually become different after manual changes. Its root filesystem is immutable, so operating-system changes are delivered as a new image rather than by editing the running system. Updates use a second disk partition: the node either starts the new version successfully or returns to the old one. Ignition defines the required initial configuration, allowing a node to be replaced consistently.

I also considered Talos Linux. Talos has an API-only administration model, which reduces the operating-system surface area further. I selected Flatcar because SSH access is useful when investigating a problem below the Kubernetes layer. This is a deliberate operational trade-off.

### RKE2

RKE2 packages Kubernetes control-plane components behind a single distribution release, reducing component-by-component upgrade coordination while retaining a controlled server-and-worker upgrade sequence. It also includes scheduled etcd snapshots, hardened defaults aligned with the Center for Internet Security (CIS) benchmark, and supported configuration for replacing kube-proxy with Cilium.

kubeadm was valuable for learning how Kubernetes components fit together. RKE2 is a better fit for a cluster that needs regular patching, because the recurring upgrade work is packaged into a defined process.

### Cilium

Cilium reduces the number of components in the cluster network. It uses extended Berkeley Packet Filter (eBPF) programs to process Kubernetes Services instead of kube-proxy and iptables rules. Its LoadBalancer IP address management (LB-IPAM) and Layer 2 announcements assign addresses to services without MetalLB. Cilium NetworkPolicy controls which workloads may communicate. Hubble shows accepted and blocked network flows. WireGuard encrypts traffic between nodes. Border Gateway Protocol (BGP) is available if the network later needs routed service addresses.

Using one supported component for these jobs means fewer independent upgrades and fewer places to investigate when networking fails.

### CloudNativePG

Databases need a clear failure model because data cannot always be recreated. Each database runs as a two-instance PostgreSQL cluster: one primary instance accepts writes and one replica receives changes. CloudNativePG promotes the replica if the primary fails.

I chose an operator instead of a PostgreSQL StatefulSet with scheduled dump scripts because CloudNativePG includes a complete database recovery model. It supports write-ahead log (WAL) archiving to object storage and recovery to a selected point in time. These features will be configured once the off-cluster backup destination described in [operations.md](operations.md) is available.

### Pulumi

Provisioning is expressed in Python because the work involves generating Ignition configuration and creating a variable number of nodes with derived addressing. Doing that in a general purpose language keeps the logic readable and testable in one place, and cluster size stays a parameter rather than copied blocks.

## Networking and web traffic

Cilium provides pod networking, network policy, WireGuard encryption, service address allocation and Layer 2 announcements. kube-vip provides the Kubernetes API address. Traefik receives HTTP traffic through a Kubernetes `LoadBalancer` Service whose address Cilium advertises on the LAN.

See [networking.md](networking.md) and [Give your bare-metal cluster real LoadBalancer IPs with Cilium](https://blog.godlyobeng.com/cilium-loadbalancer-l2-service/).

## GitOps and cluster management

Rancher provides Kubernetes management visibility, role-based access control (RBAC) through OpenID Connect, and Fleet for GitOps. Fleet watches Git repositories and applies the Kubernetes resources they contain. A rollback is therefore a Git revert followed by Fleet reconciliation.

Pulumi creates virtual machines; Fleet deploys Kubernetes workloads. Separating these responsibilities avoids two tools trying to manage the same resource.

## Storage

Longhorn is the default Kubernetes StorageClass, meaning applications receive Longhorn volumes unless they request another class. It replicates each volume across nodes. I test replica behaviour by draining nodes and checking that Longhorn rebuilds replicas. Rook and Ceph offer more storage features, but their resource requirements are not justified by this hardware.

CloudNativePG creates and manages PostgreSQL in the cluster, replacing the separate shared database virtual machine used previously. Each application declares the database it needs alongside its other Kubernetes resources. See [operations.md](operations.md).

## Identity and certificates

Keycloak provides OpenID Connect for Rancher and compatible applications, allowing access to be managed centrally. cert-manager issues certificates inside Kubernetes. step-ca is the internal certificate authority (CA) for LAN hostnames. Vaultwarden stores platform credentials.

## GPU

The NVIDIA GPU Operator manages GPU drivers and Kubernetes GPU resources. It is used for workloads that need hardware acceleration, mainly local model serving. Kubernetes schedules those workloads using the same resource controls as other applications.

## Workloads

Currently running:

| Workload | Role |
| --- | --- |
| Rancher | Kubernetes management interface and Fleet GitOps |
| Keycloak | Single sign-on and OpenID Connect provider |
| Vaultwarden | Password manager |
| Joplin | Notes, backed by CloudNativePG |
| Ghost | Blog |
| Jellyfin | Media |
| Homepage | LAN dashboard |
| Longhorn | Storage management |
| cloudflared | Outbound Cloudflare Tunnel for selected public hostnames |

I have also run GitLab, Harbor, AWX, Matrix with Synapse and MAS, Ollama, Plane, OpenProject and Moodle. Each explored a specific concern: continuous integration, container registries, automation, real-time messaging, project management or GPU scheduling. The current set is limited to services that are useful and manageable.

## Platform requirements

The following requirements guide both component choices and the work described in [operations.md](operations.md):

- A repeatable bootstrap
- A highly available Kubernetes API endpoint
- Local `LoadBalancer` addresses that do not depend on a cloud provider
- Web routing and Domain Name System (DNS) that can be investigated quickly
- Identity through OpenID Connect rather than per-application passwords
- Storage that survives losing a node
- Databases with an operator-supported recovery path
- Delivery through git, so the deployed state is auditable and reversible
- Enough network visibility to identify connection failures quickly

A component that does not support one of those is a candidate for removal.
