# Networking

All addresses in this document are examples from `192.168.1.0/24`.

## Edge

pfSense is the local area network (LAN) gateway, firewall and virtual private network (VPN) endpoint. It runs as a virtual machine on the secondary Proxmox host, separate from the host that runs most Kubernetes workloads.

The policy is deny by default, with three deliberate exceptions:

- Administrative access uses the VPN instead of an internet-exposed management port.
- Selected application hostnames use Cloudflare Tunnel, which makes an outbound connection and needs no inbound firewall rule.
- The Kubernetes API server and Kubernetes NodePort services are not reachable from the internet.

Pi-hole provides internal Domain Name System (DNS) records. A local name such as `dashboard.lab.example` resolves to a Cilium `LoadBalancer` address. Cloudflare provides public DNS for services published through its tunnel.

## Cluster network

| Item | Example | Notes |
| --- | --- | --- |
| Node LAN | `192.168.1.0/24` | The network shared by the Kubernetes nodes and other local devices. |
| Control-plane virtual IP (VIP) | `192.168.1.10` | kube-vip address used by `kubectl` and Cilium. |
| Pod Classless Inter-Domain Routing (CIDR) range | `10.244.0.0/16` | Example Pod range. With `ipam.mode: kubernetes`, this comes from the RKE2 or Kubernetes cluster configuration, not from Cilium's cluster-pool settings. |
| `LoadBalancer` pool | `192.168.1.200-240` | Addresses reserved outside the Dynamic Host Configuration Protocol (DHCP) range. |
| Traefik address | `192.168.1.200` | The address assigned to Traefik for web traffic. |

Cilium runs with `kubeProxyReplacement: true`, so it handles Kubernetes Service traffic with extended Berkeley Packet Filter (eBPF) programs instead of kube-proxy and iptables rules. Node-to-node pod traffic uses tunnel routing with Virtual Extensible LAN (VXLAN) encapsulation (`routingMode: tunnel` and `tunnelProtocol: vxlan` in the Cilium 1.19 chart). WireGuard encrypts that node-to-node traffic on the physical network.

The Cilium operator is set to one replica. The Cilium 1.19.4 chart default is two. That override is a resource trade-off for a small lab, not a claim of operator high availability. The datapath runs on each node, and existing networking continues while the operator restarts. Raise `operator.replicas` to 2 when operator availability during upgrades or node loss matters more than the extra footprint. The example values in [examples/](examples/) record the same decision.

## Service exposure

Each service uses one of three exposure methods, based on who needs to reach it.

1. **ClusterIP:** for services only used inside Kubernetes.
2. **Cilium `LoadBalancer`:** for services used by devices on the LAN, including Traefik. Cilium assigns an address from the pool and a worker node announces it using Layer 2 address resolution. Only worker nodes make these announcements, so incoming application traffic does not land on etcd control-plane nodes.
3. **Cloudflare Tunnel:** for services published to the internet. The `cloudflared` connector makes an outbound connection to Cloudflare, so the firewall does not expose an inbound port.

Kubernetes uses NodePort internally to implement some `LoadBalancer` behaviour, but applications do not depend on NodePort directly.

### Why not MetalLB

MetalLB is a well-established choice for assigning addresses to bare-metal Kubernetes services, and it worked well in the previous cluster. Cilium already handles this cluster's network traffic. Using Cilium for service addresses too removes a separate controller that would need its own configuration, upgrade path and troubleshooting process.

Details and failure modes: [Give your bare-metal cluster real LoadBalancer IPs with Cilium](https://blog.godlyobeng.com/cilium-loadbalancer-l2-service/). Example manifests: [examples/](examples/).

### Why not BGP on this LAN

The previous cluster connected Cilium to the firewall using Border Gateway Protocol (BGP) and advertised the `LoadBalancer` address range. BGP is suitable when service addresses are on a different network from the nodes, when there are several routers, or when traffic should enter through several paths.

The current cluster uses one broadcast network and a small address pool. Layer 2 announcements achieve the same result with less routing configuration. Cilium can use BGP later if the network design changes.

## Identity at the edge

Keycloak sits behind Traefik and provides OpenID Connect (OIDC) for Rancher and compatible applications. This centralises authentication and account removal. Vaultwarden stores platform credentials.

step-ca issues internal Transport Layer Security (TLS) certificates for LAN hostnames. cert-manager manages certificates inside Kubernetes. Cloudflare handles public HTTPS for hostnames that use its tunnel.

## Network policy

Cilium NetworkPolicy restricts communication around identity, web entry points and databases. For example, an application only receives network access to the database and identity services it needs. This limits what a compromised or faulty workload can reach.

The policy set is intentionally small and documented. Each rule should be understandable during an incident. Hubble shows whether Cilium allowed or blocked a connection, making policy effects visible during troubleshooting.

## Verification after a change

- `cilium status` reports healthy and confirms kube-proxy replacement is enabled.
- The `CiliumLoadBalancerIPPool` allocates addresses without reporting a conflict.
- The Traefik Service has an external address from the pool.
- A worker node holds the `cilium-l2announce-<namespace>-<service>` lease.
- A LAN client can reach the application.
- A public hostname works from outside the LAN, and a LAN hostname continues to work during an internet outage.
