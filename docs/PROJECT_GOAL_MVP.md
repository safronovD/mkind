# mkind MVP - Multi-host Kubernetes IN Docker

## Vision

mkind extends the [KinD (Kubernetes IN Docker)](https://github.com/kubernetes-sigs/kind) concept to span multiple bare-metal nodes or VMs. Where KinD runs all container "nodes" on a single host, mkind distributes them across a fleet of machines — giving you a realistic multi-host Kubernetes cluster while retaining the speed and simplicity of containerized nodes.

**Example topology:**
```
Host 1 (BM/VM)            Host 2 (BM/VM)            Host 3 (BM/VM)
+-----------------------+  +-----------------------+  +-----------------------+
| kind-cp-1  (control)  |  | kind-worker-1         |  | kind-worker-4         |
| kind-cp-2  (control)  |  | kind-worker-2         |  | kind-worker-5         |
| kind-cp-3  (control)  |  | kind-worker-3         |  | kind-worker-6         |
+-----------------------+  +-----------------------+  +-----------------------+
         \                        |                        /
          --- Docker Bridge Subnets + Static Routes -------
```

## Problem Statement

KinD is the standard tool for running local Kubernetes test clusters using Docker containers as nodes. However, it is explicitly scoped to **single-host only** (see [kind issue #1928](https://github.com/kubernetes-sigs/kind/issues/1928)). The maintainers have stated multi-machine support is out of scope.

This creates a gap: there is no tool that combines **KinD's containerized-node simplicity** with **real multi-host distribution**.

## Gap Analysis

| Requirement                                 | KinD | k3d | vind | kubeadm | k0s | **mkind** |
|---------------------------------------------|------|-----|------|---------|-----|-----------|
| Container-based K8s nodes (fast, ephemeral) | Yes  | Yes | Yes  | No      | No  | **Yes**   |
| Multi-node cluster                          | Yes  | Yes | Yes  | Yes     | Yes | **Yes**   |
| Distributed across multiple physical hosts  | No   | No  | Partial* | Yes | Yes | **Yes**   |
| Lightweight / no VM required per node       | Yes  | Yes | Yes  | No      | Partial | **Yes** |
| Uses kubeadm for bootstrap                  | Yes  | No  | No   | Yes     | No  | **Yes**   |
| Simple CLI UX (create/delete cluster)       | Yes  | Yes | Yes  | No      | Partial | **Yes** |

*vind supports joining external nodes via VPN, but the external nodes are real OS-level installs, not containerized kind-nodes.

## Existing Tools Analyzed

### KinD (kubernetes-sigs/kind)
- **What it does:** Runs K8s nodes as Docker containers on a single host, bootstrapped with kubeadm.
- **Limitation:** Single-host only. Maintainers explicitly reject multi-host scope.
- **Relevance:** mkind is built on KinD's node image and kubeadm approach. The upstream kind repo is included as a git submodule for reference.

### vind (loft-sh/vind)
- **What it does:** KinD alternative built on vCluster. Supports "Hybrid Nodes" — joining external cloud VMs via Tailscale VPN.
- **Limitation:** External nodes are real OS installs (not containerized). Requires vCluster Platform for VPN coordination. Different architecture from KinD.
- **Relevance:** Closest to mkind's vision, but fundamentally different approach (vCluster vs kubeadm, real nodes vs container nodes).

### k3d
- **What it does:** Runs k3s nodes as Docker containers. Similar concept to KinD but using k3s.
- **Limitation:** Single-host only.
- **Relevance:** Same single-host limitation as KinD.

### kubeadm-dind-cluster (kubernetes-retired)
- **What it does:** Was a kubeadm + Docker-in-Docker cluster tool. Supported remote Docker for builds.
- **Limitation:** Deprecated in favor of KinD. Cluster itself was still single-host.
- **Relevance:** Historical predecessor. Shows the DIND approach works but was never extended to multi-host.

### kubeadm (direct)
- **What it does:** The standard tool for bootstrapping multi-node K8s clusters on real machines.
- **Limitation:** Requires full OS-level setup per node. Not containerized, not ephemeral.
- **Relevance:** mkind uses kubeadm internally (via kind node images) but wraps it in containers.

### k0s
- **What it does:** Single-binary K8s distribution for bare metal / edge.
- **Limitation:** Not container-based nodes.
- **Relevance:** Alternative for real multi-host, but different UX and approach.

## Key Insight: kindnetd Works If We Solve the Underlay

After analyzing KinD's CNI plugin (`kindnetd`), the networking problem is much
simpler than initially expected.

### What kindnetd actually does

kindnetd is intentionally minimal. It does exactly two things:

1. **Adds static routes for pod CIDRs** — For each other node in the cluster,
   it runs the equivalent of `ip route add <podCIDR> via <nodeInternalIP>`.
   This is how pod-to-pod traffic crosses nodes.

2. **Writes a CNI config using the `ptp` plugin** — Each node gets a
   `10-kindnet.conflist` that uses `host-local` IPAM to assign pod IPs from
   that node's PodCIDR slice.

That's it. No VXLAN, no tunneling, no encapsulation. The **only assumption**
kindnetd makes is:

> **Every node's `InternalIP` is directly routable from every other node.**

In standard KinD this is trivially true — all containers sit on the same Docker
bridge. For mkind, if we make the kind-node container IPs routable across
hosts, kindnetd handles everything else (pod routing, CNI config, IP
masquerading) without any modification.

### The core problem reduces to one layer

```
+-----------------------------------------------------------------+
|                    What mkind must solve                         |
|                                                                 |
|   Node Underlay: make kind-node container IPs routable          |
|   across physical hosts (the Docker bridge gap)                 |
+-----------------------------------------------------------------+
        |
        v
+-----------------------------------------------------------------+
|                    What kindnetd solves for free                 |
|                                                                 |
|   Pod Overlay: static routes between nodes for pod CIDRs        |
|   CNI Config: ptp plugin + host-local IPAM per node             |
|   Masquerading: iptables rules for pod-to-external traffic      |
+-----------------------------------------------------------------+
        |
        v
+-----------------------------------------------------------------+
|                    What kube-proxy solves for free               |
|                                                                 |
|   Service Networking: ClusterIP/NodePort via iptables/IPVS      |
|   (local to each node, no cross-host routing needed)            |
+-----------------------------------------------------------------+
```

**MVP uses kindnetd exclusively.** We do NOT need to replace kindnetd with
Flannel/Calico/Cilium. We just need to solve the underlay — making
`172.18.x.x` reachable across hosts.

## Key Technical Challenges (MVP Scope)

### 1. Node Underlay Network (the core problem)

On a single host, all kind-node containers share one Docker bridge and can
reach each other directly. Across hosts, each host has its own isolated Docker
bridge — node IPs collide (`172.18.0.2` exists on every host).

**Solution: per-host unique Docker bridge subnets + static routes.**

Each host gets a unique /24 from the nodeSubnet /16 pool (e.g., Host 1 gets
`172.18.1.0/24`, Host 2 gets `172.18.2.0/24`). Cross-host routing is plain IP:

```
Host 1 (192.168.1.10)                    Host 2 (192.168.1.11)
+----------------------------+           +----------------------------+
| Docker bridge: 172.18.1.0/24 |         | Docker bridge: 172.18.2.0/24 |
| kind-cp-1:   172.18.1.2    |           | kind-worker-1: 172.18.2.2  |
| kind-cp-2:   172.18.1.3    |           | kind-worker-2: 172.18.2.3  |
+----------------------------+           +----------------------------+
         |                                          |
         +------ Same L2/L3 network (LAN) ---------+

Host 1: ip route add 172.18.2.0/24 via 192.168.1.11
Host 2: ip route add 172.18.1.0/24 via 192.168.1.10
```

No tunnels, no WireGuard, no VXLAN. Just Docker bridge networks with unique
subnets and static routes between hosts. This assumes hosts are on the same
LAN with direct L3 reachability — which is the common case for bare-metal
and VM fleets.

### 2. Coordinated Cluster Bootstrap

Need an agent or controller on each host that:
- Pulls and runs kind node images locally
- Joins them to the cluster via kubeadm join tokens
- Reports node health back to the control plane

### 3. Container Image Distribution

Kind node images need to be available on all hosts. Options:
- Pre-pull from registry on all hosts
- Shared local registry across hosts
- Image export/import via the agent

### 4. API Server Accessibility

The K8s API server must be reachable from all hosts and from the user's workstation:
- Load balancer in front of control plane nodes (like kind's haproxy)
- Port forwarding / tunnel from control plane host

## High-Level Architecture

```
                          User Workstation
                               |
                          mkind CLI
                               |
                    +----------+----------+
                    |                     |
              SSH / gRPC            SSH / gRPC
                    |                     |
           +--------+--------+   +--------+--------+
           |   Host 1        |   |   Host 2        |
           |  mkind-agent    |   |  mkind-agent    |
           |  +------------+ |   |  +------------+ |
           |  | kind-node  | |   |  | kind-node  | |
           |  | kind-node  | |   |  | kind-node  | |
           |  | kind-node  | |   |  | kind-node  | |
           |  +------------+ |   |  +------------+ |
           +-----------------+   +-----------------+
                    |                     |
                    +--- Static Routes ----+
```

**Components:**
- **mkind CLI** — User-facing tool. Reads cluster config, orchestrates creation across hosts.
- **mkind-agent** — Runs on each host. Manages local Docker containers (kind nodes), handles kubeadm join, reports status.
- **Cluster config** — YAML defining hosts, node placement, networking, and K8s version.

## Example Config (Target UX)

```yaml
apiVersion: mkind.io/v1alpha1
kind: Cluster
metadata:
  name: my-cluster
spec:
  kubernetesVersion: v1.31.0
  networking:
    # Underlay mode: how kind-node container IPs are routed across hosts.
    # This is the ONE thing mkind must solve — everything else (pod routing,
    # CNI config, service IPs) is handled by kindnetd and kube-proxy for free.
    mode: bridge  # Docker bridge + static routes (MVP)
    # Node subnet: IP range for kind-node containers themselves.
    # Each host gets a unique /24 slice (auto-allocated).
    # This is the underlay — kubelets, etcd, API server use these IPs.
    # kindnetd's only requirement: these must be routable across all hosts.
    nodeSubnet: 172.18.0.0/16
    # Pod subnet: IP range for Kubernetes pods.
    # Managed by kindnetd (static routes) + ptp CNI plugin.
    # No extra CNI needed — kindnetd handles this once the underlay works.
    podSubnet: 10.244.0.0/16
    # Service subnet: virtual IPs for Kubernetes Services.
    # Handled locally by kube-proxy (iptables/IPVS). Not routed on the wire.
    serviceSubnet: 10.96.0.0/12
  hosts:
    - address: 192.168.1.10
      user: ubuntu
      # Auto-allocated from nodeSubnet: 172.18.1.0/24
      nodes:
        - role: control-plane
        - role: control-plane
        - role: control-plane
    - address: 192.168.1.11
      user: ubuntu
      # Auto-allocated from nodeSubnet: 172.18.2.0/24
      nodes:
        - role: worker
        - role: worker
        - role: worker
    - address: 192.168.1.12
      user: ubuntu
      # Auto-allocated from nodeSubnet: 172.18.3.0/24
      nodes:
        - role: worker
        - role: worker
        - role: worker
```

## Target CLI UX

```bash
# Create a cluster from config
mkind create cluster --config cluster.yaml

# List clusters
mkind get clusters

# Get nodes across all hosts
mkind get nodes

# Delete a cluster
mkind delete cluster my-cluster

# Install agent on a remote host
mkind agent install --host 192.168.1.10 --user ubuntu
```

## Technology Choices

- **Language:** Go (same as KinD, native K8s ecosystem)
- **License:** Apache 2.0 (same as KinD and K8s projects)
- **Upstream reference:** KinD repo included as git submodule at `upstream/kind`
- **Bootstrap:** kubeadm (via kind node images)
- **CLI framework:** cobra
- **Cross-host communication:** SSH (initial) / gRPC (future)
- **CNI:** kindnetd (unmodified from KinD — works if the underlay is flat)
- **Underlay networking:** Docker bridge with per-host subnets + static routes

## MVP Milestones

### Phase 1 — Foundation
- [ ] Project scaffolding (Go module, CLI skeleton, CI)
- [ ] Cluster config schema (v1alpha1)
- [ ] SSH-based agent that can run kind node containers on a remote host
- [ ] Single control-plane creation across 2 hosts

### Phase 2 — Multi-Node
- [ ] HA control plane (3 CP nodes across hosts)
- [ ] Worker node join across hosts
- [ ] Cross-host node underlay with Docker bridge + static routes (kindnetd handles pod routing)
- [ ] Image preloading across hosts
