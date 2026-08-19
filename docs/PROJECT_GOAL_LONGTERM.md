# mkind Long-Term Goals

Post-MVP goals for mkind. These build on the working MVP (kindnetd + Docker
bridge with static routes) and expand the project into a more complete platform.

## Alternative CNI Support

The MVP uses kindnetd exclusively. Once the underlay works and clusters are
stable, a natural next step is supporting pluggable CNI plugins. This enables
users to test their workloads with the same CNI they run in production.

### Target CNIs to integrate

| CNI | Why | Notes |
|---|---|---|
| **Cilium** | eBPF-based, widely adopted, replaces kube-proxy | Requires kernel 4.19+, may need custom kind node images |
| **Calico** | Most popular production CNI, BGP and VXLAN modes | Can run in overlay or native routing mode |
| **Flannel** | Simple, widely used, VXLAN-based | Closest to kindnetd in simplicity |
| **Weave Net** | Encrypted mesh, no external deps | Good for edge / air-gapped scenarios |
| **Antrea** | VMware-backed, OVS-based | Good for enterprise / VMware shops |

### Implementation approach

1. **Disable kindnetd** — Pass `--networking.disableDefaultCNI=true` (or
   equivalent kubeadm config) during cluster bootstrap so kindnetd is not
   deployed.

2. **CNI installer** — After k8s-nodes join, deploy the chosen CNI via its standard
   manifest (e.g., `kubectl apply -f cilium.yaml`). This is how production
   clusters install CNIs.

3. **Config extension** — Add a `cni` field to the cluster config:
   ```yaml
   spec:
     networking:
       cni: cilium  # or: calico, flannel, weave, kindnetd (default)
       cniConfig:
         # CNI-specific options
         cilium:
           kubeProxyReplacement: true
           hubble: true
   ```

4. **Underlay interaction** — Some CNIs (Calico BGP, Cilium native routing)
   expect flat L2/L3 between k8s-nodes. The MVP's Docker bridge + static routes
   underlay provides this naturally. If using tunnel-based underlays (see
   below), may need MTU tuning to account for double encapsulation.

### Considerations

- **kindnetd stays the default** — It is the simplest option and requires zero
  configuration beyond the underlay. Alternative CNIs are opt-in.
- **K8s-node image compatibility** — Some CNIs need kernel modules or eBPF support
  that may not be present in the default kind node image. We may need to build
  custom images or document requirements.
- **Testing matrix** — Each CNI adds a testing dimension. Start with one
  alternative (Cilium, given its momentum) and expand.

## CNI Performance Comparison

A key value-add for mkind: run the same workload across different CNIs on
identical multi-host topologies and compare performance. This is hard to do
today because setting up identical multi-k8s-node clusters with different CNIs is
tedious.

### Metrics to capture

| Metric | Tool | What it measures |
|---|---|---|
| Pod-to-pod throughput | iperf3 / netperf | Raw bandwidth between pods on different hosts |
| Pod-to-pod latency | netperf (TCP_RR) / qperf | Round-trip time for small messages |
| Service (ClusterIP) latency | custom benchmark | Overhead of kube-proxy / eBPF service routing |
| Network policy enforcement overhead | iperf3 with/without policies | Cost of applying NetworkPolicy rules |
| Connection setup rate | netperf (TCP_CRR) | New connections per second |
| DNS resolution latency | dig / dnsperfgo | CoreDNS + CNI interaction |
| CPU overhead on hosts | top / perf / eBPF profiling | CPU cost of CNI dataplane per Gbps |

### Benchmark workflow

```bash
# Create identical clusters with different CNIs
mkind create cluster --config cluster.yaml --set networking.cni=kindnetd --name bench-kindnetd
mkind create cluster --config cluster.yaml --set networking.cni=cilium    --name bench-cilium
mkind create cluster --config cluster.yaml --set networking.cni=calico    --name bench-calico

# Run benchmark suite against each
mkind benchmark run --cluster bench-kindnetd --output results/kindnetd.json
mkind benchmark run --cluster bench-cilium   --output results/cilium.json
mkind benchmark run --cluster bench-calico   --output results/calico.json

# Compare results
mkind benchmark compare results/*.json
```

### Benchmark suite design

- Deploy iperf3/netperf server and client pods on specific k8s-nodes (cross-host)
- Run standardized tests with consistent parameters
- Collect results in structured format (JSON)
- Generate comparison tables and charts
- Document test methodology for reproducibility

## Production Readiness (Phase 3)

- [ ] gRPC agent with health reporting
- [ ] K8s-node placement constraints and affinity
- [ ] Cluster lifecycle (upgrade, scale, backup)
- [ ] Resource management (CPU/memory awareness, scheduling)
- [ ] Documentation and examples

## Multi-Tenancy: Multiple Clusters Per Host

The MVP supports **one mkind cluster per host** — a host participating in a
cluster cannot join a second one, and attempting it is refused rather than
left undefined.

Long-term, several clusters should be able to share the same subset of hosts
concurrently. The driving case is CI: parallel jobs on a shared lab pool
should not have to serialize on host availability, and a lab operator should
be able to run a stable cluster alongside a throwaway one on the same
machines.

What this requires beyond the MVP:

- [ ] Cluster-scoped naming for every host resource (containers, bridges,
      routes, persisted units) so cleanup can never touch a sibling cluster
- [ ] K8s-node subnet allocation that accounts for ranges already claimed by
      other mkind clusters on the same host, not just within one cluster
- [ ] Host port arbitration for each cluster's API endpoint
- [ ] `mkind get clusters` reporting several clusters per host, and
      per-cluster filtering throughout the CLI
- [ ] Concurrent-create safety: two clusters being created against the same
      host at once must not interleave allocations

## Fully Idempotent `create`

The MVP treats `create` as best-effort convergent: re-running it against an
existing cluster with an unchanged config will skip healthy k8s-nodes and
provision missing ones where it can, but makes no promise. The guaranteed
recovery path after a partial failure is `delete` then `create`.

Long-term, `create` should be a true reconciler — compare desired state to
observed state per resource and converge, every time, with no teardown.

- [ ] Per-resource desired-vs-observed comparison (containers, subnets,
      routes, persisted units, key material, kubeconfig entries)
- [ ] Converge a cluster whose config changed rather than refusing it
      (FR-036 today) — add/remove k8s-nodes in place
- [ ] Repair a k8s-node that exists but is unhealthy, instead of only
      provisioning ones that are missing
- [ ] Dry-run / diff output showing what a `create` would change

## Config-Free Cluster Discovery (`mkind get clusters`)

In the MVP, `mkind get clusters` and `mkind get nodes` require a cluster
config naming the hosts to query — mkind has no workstation-side state, so
without it there is nothing to point at.

Upstream kind needs no such argument: it asks the local Docker daemon for
containers carrying the `io.x-k8s.kind.cluster` label (with
`io.x-k8s.kind.role` distinguishing control-plane from worker) and derives
the cluster list from what comes back. No state file, single source of
truth, nothing to go stale.

mkind should match that UX. The MVP already labels k8s-node containers the same
way, so per-host discovery works — **the open problem is host inventory**:
kind has exactly one daemon to ask, mkind has N, and knowing which N is
precisely the state kind never has to keep.

- [ ] **Research upstream kind's provider/discovery code** (`pkg/cluster/internal/providers`)
      to see how far its model stretches to multiple daemons
- [ ] Decide where the host inventory lives without reintroducing a state
      store that can drift from reality — candidates: a well-known config
      directory of previously used host sets, a marker on the hosts
      themselves that a discovery pass can sweep, or an explicit
      `mkind config add-host` registration
- [ ] `mkind get clusters` with no arguments listing every cluster across
      all known hosts, matching `kind get clusters` output
- [ ] Decide the failure mode when a known host is unreachable during a
      listing: partial results with a warning, or hard failure

## Ecosystem (Phase 4)

- [ ] CI/CD integration (GitHub Actions, etc.)
- [ ] Terraform / Pulumi provider
- [ ] Web UI for cluster management

## Tunnel-Based Underlay Options

The MVP uses Docker bridge + static routes, which requires hosts on the same
LAN. For hosts across different networks, NAT boundaries, or the internet,
add tunnel-based underlay modes:

| Approach | How it works | Pros | Cons |
|---|---|---|---|
| **WireGuard** | Point-to-point encrypted tunnels between hosts, route k8s-node subnets through them | Fast (kernel-space), encrypted, simple config | Requires WireGuard on each host |
| **VXLAN (manual)** | Create VXLAN interfaces on each host, bridge to Docker network | No extra deps, production-like | Most manual setup |
| **Docker Swarm overlay** | `docker network create --driver overlay --attachable` across Swarm hosts | Docker-native, automatic VXLAN | Requires Swarm init, extra abstraction layer |
| **Tailscale** | Managed WireGuard via Tailscale coordination server | Zero-config NAT traversal | External dependency (Tailscale account) |
| **Nebula** | Open-source overlay network (Slack's fork of WireGuard concepts) | Self-hosted, no external deps | Less mature than WireGuard |
