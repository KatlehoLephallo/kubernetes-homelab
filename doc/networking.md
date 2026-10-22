# Networking

This document is the source of truth for every address used in the homelab.
Anything that allocates an IP (Ansible inventory, libvirt network definitions,
kubeadm config, CNI config) must match what is recorded here.

## Overview

Two mini PCs run KVM/libvirt and connect to the home router over Wi-Fi only.
Wi-Fi interfaces can't be bridged, so each host gets its own **routed** libvirt
network with a dedicated VM subnet, instead of putting VMs directly on the home LAN.


```mermaid
---
title: Kubernetes Homelab — Network Topology
---
flowchart TB
    router["<b>Home Router</b><br/>192.168.18.0/24"]
 
    subgraph kvm1["<b>KVM1 · 192.168.18.201</b>"]
        
        direction TB
        gw1["<b>libvirt Virtual Gateway</b><br/>10.10.1.1"]
        subgraph net1["<b>VM Subnet · 10.10.1.0/24</b>"]
            direction LR
            w1["<b>Worker 1</b><br/> 10.10.1.11"]
            w2["<b>Worker 2</b><br/> 10.10.1.12"]
        end
        gw1 --> net1
    end
 
    subgraph kvm2["<b>KVM2 · 192.168.18.202</b>"]
        direction TB
        gw2["<b>libvirt Virtual Gateway</b><br/>10.10.2.1"]
        subgraph net2["<b>VM Subnet · 10.10.2.0/24</b>"]
            cp1["<b>Control Plane</b><br/> 10.10.2.10"]
        end
        gw2 --> net2
    end
 
    router -. "Wi-Fi" .- kvm1
    router -. "Wi-Fi" .- kvm2
    kvm1 <-. "Static routes" .-> kvm2
 
    classDef routerStyle fill:#1e293b,stroke:#64748b,stroke-width:2px,color:#f1f5f9
    classDef gatewayStyle fill:#3d2f1c,stroke:#f0b54a,stroke-width:2px,color:#f1f5f9
    classDef vmStyle fill:#163a30,stroke:#34d399,stroke-width:2px,color:#f1f5f9
 
    class router routerStyle
    class gw1,gw2 gatewayStyle
    class w1,w2,cp1 vmStyle
 
    style kvm1 fill:#111c2e,stroke:#64748b,stroke-width:2px,color:#f1f5f9
    style kvm2 fill:#111c2e,stroke:#64748b,stroke-width:2px,color:#f1f5f9
    style net1 fill:#0f2e2a,stroke:#2dd4bf,stroke-width:2px,color:#f1f5f9
    style net2 fill:#0f2e2a,stroke:#2dd4bf,stroke-width:2px,color:#f1f5f9
 
    linkStyle 0,1 stroke:#f0b54a,stroke-width:2px
    linkStyle 2,3,4 stroke:#60a5fa,stroke-width:2px
```


## Physical hosts

| Host | IP |  CPU | RAM |
|------|----|-----|-----|
| kvm1 | 192.168.18.201 |  24 vCPU  | 27 GiB |
| kvm2 | 192.168.18.202 |  12 vCPU  | 28 GiB |

## VM networks (libvirt, routed mode)

| Host | Subnet | Gateway (host bridge) |
|------|--------|-----------------------|
| kvm1 | 10.10.1.0/24 | 10.10.1.1 |
| kvm2 | 10.10.2.0/24 | 10.10.2.1 |

## Cluster nodes

| Node | Role | Host | IP |
|------|------|------|----|
| cp | control plane | kvm2 | 10.10.2.10 |
| worker1 | worker | kvm1 | 10.10.1.11 |
| worker2 | worker | kvm1 | 10.10.1.12 |



## Kubernetes ranges

| Range | CIDR | Notes |
|-------|------|-------|
| Pods | 10.244.0.0/16 | Set explicitly in kubeadm (`podSubnet`) and the CNI config |
| Services | 10.96.0.0/12 | Set explicitly in kubeadm (`serviceSubnet`) |

Both ranges are set explicitly rather than left to defaults. Calico's default
pod CIDR (192.168.0.0/16) would overlap the home LAN.

## DNS

- **Upstream resolvers for VMs:** 1.1.1.1 and 8.8.8.8 (two providers for redundancy)
- **Node names:** written to `/etc/hosts` on every VM by Ansible, from the inventory
- **Inside the cluster:** CoreDNS serves Service names and forwards everything else
  to the node's resolvers. Pods can't resolve node hostnames by default.

See the DNS ADR for the alternatives considered.

## Overlap check

None of these ranges overlap:

| Range | Used for |
|-------|----------|
| 192.168.18.0/24 | Home LAN |
| 10.10.1.0/24 | kvm1 VMs |
| 10.10.2.0/24 | kvm2 VMs |
| 10.244.0.0/16 | Pods |
| 10.96.0.0/12 | Services (10.96.0.0 – 10.111.255.255) |






## Known follow-ups

- **Host routes:** each host needs a route to the other's VM subnet
  (kvm1: 10.10.2.0/24 via 192.168.18.202; kvm2: 10.10.1.0/24 via 192.168.18.201),
  because the router doesn't support static routes.
- **VM internet egress:** the router can't route replies to 10.10.x.x, so each host must
  masquerade VM traffic leaving for the internet. Exclude these destinations from NAT:
  10.10.0.0/16, 10.244.0.0/16, 10.96.0.0/12.
