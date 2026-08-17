# Example manifests

These are redacted examples of the configuration patterns described elsewhere in this repository. The addresses are examples from `192.168.1.0/24`. Before using them, replace the network range, Dynamic Host Configuration Protocol (DHCP) range, network interface names and Kubernetes API address with values for your own environment.

| File | Purpose |
| --- | --- |
| [cilium-values.example.yaml](cilium-values.example.yaml) | Cilium 1.19 Helm values for kube-proxy replacement, Layer 2 announcements and WireGuard. With `ipam.mode: kubernetes`, set the Pod CIDR in the RKE2 or Kubernetes cluster configuration, not under `ipam.operator`. `operator.replicas: 1` is an intentional override of the chart default of 2. |
| [CiliumLoadBalancerIPPool.example.yaml](CiliumLoadBalancerIPPool.example.yaml) | The local area network (LAN) address range for Kubernetes `LoadBalancer` Services |
| [CiliumL2AnnouncementPolicy.example.yaml](CiliumL2AnnouncementPolicy.example.yaml) | Opt-in Address Resolution Protocol (ARP) announcements from worker nodes |
| [helmfile.example.yaml](helmfile.example.yaml) | Helmfile configuration that keeps Helm values under version control |

Longer explanation: [Give your bare-metal cluster real LoadBalancer IPs with Cilium](https://blog.godlyobeng.com/cilium-loadbalancer-l2-service/).
