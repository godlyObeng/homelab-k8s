# 🧪 Homelab Kubernetes Cluster

Welcome to my Homelab Kubernetes setup! This repo documents my self-hosted Kubernetes environment, which I've designed for learning, testing, and self-hosting various cloud-native applications. Below is an overview of the hardware, network topology, and software stack that powers the setup.

![Homelab Kubernetes Diagram](https://github.com/godlyObeng/homelab-k8s/blob/44699777b03984ac8105350e0e89493e24fcbf45/Screenshot%202025-04-14%20at%2018.00.24.png)

## 🖥️ Hardware Setup

The cluster is built using three HP ProDesk SFF machines:

- **CPU**: Intel i5 6500 (4 cores)
- **RAM**: 32GB
- **Storage**: 256GB SSD + 2TB HDD
- **OS**: Rocky Linux 9.5

## 🌐 Network Design

- **External**: All nodes are connected to a 1GbE external switch which connects to the internet through an **OPNsense firewall**.
- **Internal Ring**: A dedicated **2.5Gbps internal ring network** connects the nodes directly for fast east-west traffic and Cilium's overlay network.
- **Routing**: BGP is enabled between the Kubernetes cluster and OPNsense to expose services natively without ingress overlays.

### Ring Network Connections

| Node            | enp4s0 (out) | enp5s0 (in) |
|-----------------|-------------|-------------|
| hp-cluster-01   | → 03        | ← 02        |
| hp-cluster-02   | → 01        | ← 03        |
| hp-cluster-03   | → 02        | ← 01        |

## 🧠 Software Services

This cluster runs the following services:

- **[Longhorn](https://longhorn.io/)** – Distributed block storage for Kubernetes.
- **[Cilium](https://cilium.io/)** – CNI with BPF-based networking, security, and BGP support.
- **[CloudNativePG](https://cloudnative-pg.io/)** – PostgreSQL operator for managing highly available DBs.
- **[Harbor](https://goharbor.io/)** – Container image registry.
- **[Keycloak](https://www.keycloak.org/)** – Identity and access management.
- **[AWX](https://github.com/ansible/awx)** – Ansible web UI for automation.
- **[Rancher](https://rancher.com/)** – Kubernetes management platform.
- **[NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)** – Ingress traffic routing.

## 🔐 Security & Access

- All traffic between nodes uses the 2.5Gbps internal network.
- Cilium handles network policy enforcement.
- Services are exposed externally via BGP and optionally proxied via NGINX or Cloudflare.

## 📦 Deployment Tools

- Cluster bootstrapped with `kubeadm`.
- Infrastructure automation via Ansible (AWX for orchestration).
- GitOps style CI/CD and monitoring coming soon.
