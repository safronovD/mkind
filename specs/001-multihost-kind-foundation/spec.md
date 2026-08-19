# Feature Specification: Multi-host KinD Foundation and Multi-node Bootstrap (MVP Phase 1–2)

**Feature Branch**: `usr/safronovD/5-mvp-phase-1-2-foundation`

**Created**: 2026-07-16

**Updated**: 2026-08-19

**Status**: Draft

**Input**: User description: "Implement MVP Phase 1–2 as described in docs/PROJECT_GOAL_MVP.md: Phase 1 — Foundation (project scaffolding, cluster config schema v1alpha1, remote agent that can run containerized cluster nodes on a remote host, single control-plane creation across 2 hosts). Phase 2 — Multi-Node (HA control plane with 3 control-plane nodes, worker node join across hosts, cross-host node connectivity, image preloading across hosts)." (GitHub issue #5)

## Clarifications

### Session 2026-08-19

- Q: How are SSH credentials supplied, and how is host-to-host access authenticated given that reboot recovery must run unattended? → A: Passwordless key-based SSH from the workstation to every host is an operator-configured prerequisite; mkind never handles passwords. The user's own identity is used for workstation→host only and is never copied to a host. For host→host, mkind generates a dedicated per-cluster keypair at create time, installs it with restricted `authorized_keys` entries, persists it across reboots, and removes it on cluster deletion.
- Q: What is the recovery path after a create that partially failed? → A: Re-running `create` against the same cluster and config SHOULD converge it on a **best-effort** basis — skipping healthy k8s-nodes, provisioning missing ones — but this is not a guarantee. The guaranteed, supported recovery path is `delete` followed by `create`, which deterministic allocation makes reproducible. Where convergence is not possible, creation must fail clearly and tell the user to delete and recreate, never partially mangle an existing cluster.
- Q: How is the kubeconfig delivered to the workstation? → A: Matching upstream kind. The administrator credentials are read from a control-plane k8s-node's kubeadm-generated admin config, the `server` address is rewritten from the in-cluster address to the cluster's stable externally reachable endpoint, and the result is merged into the user's default kubeconfig as context `mkind-<cluster>` and set current. `--kubeconfig` overrides the target file, an `export kubeconfig` verb regenerates it, and `delete` removes the context again.
- Q: With no state store, what do the `get` commands inspect to discover clusters? → A: In the MVP the user supplies the cluster config, and mkind queries exactly the hosts named in it, discovering resources by container label the way `kind get clusters` does per daemon. Config-free discovery matching upstream kind's UX needs host-inventory research and is deferred to `docs/PROJECT_GOAL_LONGTERM.md`.
- Q: Can a single host participate in more than one mkind cluster at the same time? → A: Not in the MVP — one cluster per host. It is a wanted capability and is recorded as future work in `docs/PROJECT_GOAL_LONGTERM.md`, but MVP creation refuses a host that already belongs to a cluster rather than leaving the behavior undefined.

## Terminology *(canonical — used consistently throughout)*

mkind spans two kinds of machine, and the word "node" alone is ambiguous between them. This document never uses it bare where the meaning could be either.

| Term | Means | Never means |
|---|---|---|
| **host** (or **host-node**) | A physical or virtual *machine* participating in a cluster — a bare-metal box or a VM, reachable over SSH, running a container runtime. | A Kubernetes node |
| **k8s-node** (or **kubernetes-node**) | A *containerized Kubernetes node* — one container running kubelet, registered with the cluster, holding a role of control-plane or worker. Several run on one host. | A physical machine |

- "control-plane node" and "worker node" are **k8s-nodes**; the role prefix makes them unambiguous.
- "k8s-node address" and "k8s-node subnet" are the underlay addresses of k8s-node *containers* — not the hosts' own LAN addresses.
- Kubernetes-standard phrasing such as "all k8s-nodes report Ready" refers to k8s-nodes, matching `kubectl get nodes`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create a cluster on a single host (Priority: P1)

A Kubernetes practitioner writes a declarative configuration file listing exactly one remote host with a control-plane node and one or more worker nodes on it. They run one command from their workstation, and the tool provisions k8s-nodes on that host, bootstraps a working Kubernetes cluster, and hands back credentials usable from the workstation.

**Why this priority**: This is the smallest end-to-end vertical slice of the product. It exercises everything the multi-host path needs — config parsing and validation, remote host access, k8s-node provisioning, cluster bootstrap, and credential delivery — with cross-host routing removed as a variable. It is also the only configuration that can be exercised on a single CI runner, so it becomes the automated regression harness for every later story. Multi-host is the product's reason to exist, but it is not the first thing that should work.

**Independent Test**: Can be fully tested by preparing one reachable Linux host, writing a config placing a control-plane node and a worker node on it, running the create command, and verifying both k8s-nodes report Ready and the cluster is manageable from the workstation.

**Acceptance Scenarios**:

1. **Given** one prepared host and a valid single-host config, **When** the user runs the cluster create command, **Then** a cluster is created with all k8s-nodes in Ready state and the user receives working access credentials.
2. **Given** a config file with invalid content (unknown fields, missing required values, malformed addresses), **When** the user runs the create command, **Then** the command fails before touching any host, with an error message identifying the exact problem and its location in the config.
3. **Given** a single-host cluster was created, **When** the user points their standard Kubernetes client at the returned credentials, **Then** they can list k8s-nodes and deploy workloads that reach Ready.

---

### User Story 2 - Create a cluster spanning two hosts from a config file (Priority: P1)

A Kubernetes practitioner writes a single declarative configuration file describing a cluster: which remote hosts participate, how many k8s-nodes run on each host, and each k8s-node's role. They run one command from their workstation, and the tool provisions k8s-nodes on each remote host, bootstraps a working Kubernetes cluster with a single control-plane node on one host and a worker node on another host, and hands back credentials so the practitioner can immediately manage the cluster with their usual tooling.

**Why this priority**: This is the core value proposition — a multi-host containerized Kubernetes cluster from one command. It adds the one thing Story 1 deliberately left out: cross-host k8s-node reachability for the worker-to-control-plane connection.

**Independent Test**: Can be fully tested by preparing two reachable Linux hosts, writing a config with one control-plane node on host A and one worker node on host B, running the create command, and verifying both k8s-nodes report Ready and the cluster is manageable from the workstation.

**Acceptance Scenarios**:

1. **Given** two prepared hosts that satisfy the network requirements below — each reachable from the workstation over passwordless key-based SSH, and each able to open an SSH connection to the other — and a valid config placing one control-plane node on host A and one worker node on host B, **When** the user runs the cluster create command, **Then** a cluster is created with both k8s-nodes in Ready state and the user receives working access credentials.
2. **Given** a config naming a host that the workstation can reach but the peer host cannot reach over SSH, **When** the user runs the create command, **Then** preflight fails identifying the unreachable host pair and the reason, before any resources are created.
3. **Given** a host that accepts only password-based SSH authentication, **When** the user runs the create command, **Then** preflight fails naming that host and stating that passwordless key-based SSH must be configured first, before any resources are created.
4. **Given** a cluster was created, **When** the user points their standard Kubernetes client at the returned credentials from their workstation, **Then** they can list k8s-nodes and observe both k8s-nodes across the two hosts.
5. **Given** the same config file, **When** the cluster is deleted and recreated, **Then** the resulting topology and k8s-node addressing are identical (deterministic allocation).

---

### User Story 3 - Workloads run and communicate across hosts (Priority: P2)

After creating a multi-host cluster, the practitioner deploys workloads and expects them to behave exactly as on a conventional cluster: pods are scheduled onto k8s-nodes regardless of which physical host the k8s-node lives on, pods on different hosts can talk to each other directly, and cluster-internal service addresses work from any k8s-node.

**Why this priority**: A cluster where cross-host pods cannot communicate is not a usable cluster — but this capability builds on Story 2's cluster creation, so it is sequenced after it. It represents the key technical differentiator: making k8s-node addresses reachable across physical host boundaries so standard Kubernetes networking works unmodified.

**Independent Test**: Can be tested by creating a two-host cluster, deploying one pod pinned to a k8s-node on each host, and verifying direct pod-to-pod communication and service-based communication in both directions.

**Acceptance Scenarios**:

1. **Given** a running two-host cluster, **When** a pod on host A sends traffic to a pod on host B by pod address, **Then** the traffic arrives and a reply returns.
2. **Given** a running two-host cluster with a cluster-internal service backed by pods on both hosts, **When** any pod calls the service address, **Then** requests succeed regardless of which host serves them.
3. **Given** a workload with no placement constraints, **When** it is scaled to more replicas than one host's k8s-nodes can hold, **Then** the scheduler places replicas on k8s-nodes across multiple hosts and all replicas become ready.

---

### User Story 4 - Highly available control plane co-located on one host (Priority: P2)

A practitioner testing production-like topologies defines a cluster with an odd number of control-plane nodes (3 or 5) **all placed on the same host**, plus worker nodes on other hosts. The tool bootstraps every control-plane node into a single coordinated control plane, joins the workers from the other hosts, and provides a single stable access endpoint for the cluster.

**Why this priority**: Multi-member control-plane topologies are a primary reason practitioners need multi-host clusters — they want many workers on separate machines behind a realistic control plane. Keeping every control-plane member on one host is a deliberate MVP constraint: it keeps the quorum's peer traffic entirely on one machine's local bridge, so cross-host latency and partitions cannot corrupt the datastore quorum, and the cross-host requirement stays limited to worker-to-control-plane reachability (Story 3). It extends Story 2's bootstrap and depends on Story 3's cross-host reachability.

**Independent Test**: Can be tested by creating a cluster from a config with three control-plane nodes on host A and at least two workers on host B, then verifying all k8s-nodes reach Ready and the cluster is reachable through one endpoint.

**Acceptance Scenarios**:

1. **Given** a config with three control-plane nodes on one host and workers on another host, **When** the user creates the cluster, **Then** all k8s-nodes reach Ready state and cluster management operations work from the workstation.
2. **Given** a running multi-member cluster, **When** the user inspects k8s-nodes, **Then** all three control-plane nodes are listed as control-plane members of one cluster.
3. **Given** a config with five control-plane nodes on one host, **When** the user creates the cluster, **Then** the same outcome holds.
4. **Given** a config that places control-plane nodes on more than one host, **When** the user runs the create command, **Then** validation rejects it before contacting any host, naming the hosts that hold conflicting control-plane placements.
5. **Given** a config with an even number of control-plane nodes, **When** the user runs the create command, **Then** validation rejects it, stating that control-plane counts must be odd.

---

### User Story 5 - Cluster self-heals after host reboots (Priority: P2)

A practitioner runs clusters on lab or CI machines that get rebooted — kernel patching, scheduled maintenance, power loss. After each participating host comes back online, they expect the cluster to return to a fully working state on its own: no re-running create, no manual container starts, no re-adding routes.

**Why this priority**: On the target hardware, reboots are routine rather than exceptional, and a cluster that must be recreated after every reboot is not usable for the long-lived lab and CI scenarios this product exists to serve. It applies to every topology from Story 1 onward, so it is a peer of the networking and HA stories rather than a later polish item.

**Independent Test**: Can be tested by creating a multi-host cluster, rebooting one host, waiting for it to come up, and verifying without running any mkind command that all k8s-nodes return to Ready — then repeating with the control-plane host, and then with all hosts at once.

**Acceptance Scenarios**:

1. **Given** a running multi-host cluster, **When** a worker host is rebooted and comes back online, **Then** its k8s-node containers start automatically, its k8s-nodes rejoin, and the cluster returns to all-k8s-nodes-Ready with no user action.
2. **Given** a running multi-host cluster, **When** the host holding the control plane is rebooted, **Then** the control plane comes back on the same endpoint, the previously issued credentials keep working unchanged, and all k8s-nodes across all hosts return to Ready.
3. **Given** a running multi-host cluster, **When** every host is rebooted at once (lab power cycle), **Then** the cluster reconverges to all-k8s-nodes-Ready once the hosts are up, with no user action.
4. **Given** a host has been rebooted, **When** the cross-host network configuration on that host is inspected, **Then** the per-host k8s-node subnet, the static routes to peer hosts' k8s-node subnets, and packet forwarding are all still in effect.
5. **Given** a host has been rebooted, **When** the cluster reconverges, **Then** every k8s-node on that host holds the same address it had before the reboot.
6. **Given** a rebooted host does not come back within a bounded time, **When** the user lists the cluster's nodes, **Then** the affected k8s-nodes are reported NotReady with their host identified, rather than failing silently.

---

### User Story 6 - Inspect and delete clusters cleanly (Priority: P3)

A practitioner managing test infrastructure lists existing clusters, views the k8s-nodes of a cluster together with which host each node runs on, and deletes a cluster when finished. Deletion removes every resource the tool created on every host — k8s-node containers, network configuration, routing entries, and anything installed to make them survive a reboot — leaving hosts as they were.

**Why this priority**: Lifecycle hygiene is essential for repeated use (CI, shared lab hosts) but only matters once clusters can be created. Incomplete cleanup would poison subsequent runs — and Story 5's reboot persistence makes complete cleanup strictly harder, since leftovers would now come back after a reboot.

**Independent Test**: Can be tested by creating a cluster, listing it, deleting it, verifying no tool-created resources remain on any host (including after rebooting a host), and then recreating a cluster with the same name successfully.

**Acceptance Scenarios**:

1. **Given** one or more running clusters, **When** the user runs the list command against a config naming their hosts, **Then** all clusters found on those hosts are shown by name.
2. **Given** a running cluster, **When** the user runs the k8s-node listing command, **Then** every k8s-node is shown with its role, status, and the host it runs on.
3. **Given** a running cluster, **When** the user deletes it by name, **Then** all k8s-node containers, network configuration, routing entries, reboot-persistence settings, and per-cluster key material created by the tool are removed from every participating host, and the cluster's kubeconfig context is removed while every other context in that file is left intact.
4. **Given** a cluster was deleted, **When** a participating host is rebooted, **Then** no k8s-node containers or routes from the deleted cluster reappear.
5. **Given** a cluster was deleted, **When** the user creates a new cluster with the same name and config, **Then** creation succeeds without manual cleanup.

---

### User Story 7 - Preload k8s-node images across hosts (Priority: P3)

A practitioner working with slow networks, repeated CI runs, or restricted environments distributes the required k8s-node images to all participating hosts ahead of cluster creation, so that creating a cluster does not depend on each host independently downloading images at creation time.

**Why this priority**: This accelerates and hardens cluster creation but is an optimization — creation must already work (Stories 1–4) for preloading to add value.

**Independent Test**: Can be tested by preloading images to hosts, then creating a cluster while external image downloads are unavailable, and verifying creation succeeds.

**Acceptance Scenarios**:

1. **Given** prepared hosts without the required k8s-node images, **When** the user runs the preload operation, **Then** all required images become available on every configured host.
2. **Given** hosts with preloaded images, **When** the user creates a cluster without access to external image sources, **Then** creation succeeds using the local images.
3. **Given** hosts with preloaded images, **When** the user creates a cluster, **Then** no host re-downloads images that are already present.

---

### Edge Cases

- A host listed in the config is unreachable from the workstation or rejects the SSH identity: creation fails fast during preflight with the host address and the reason, before any resources are created elsewhere.
- A host offers only password-based SSH, or the named SSH identity is missing, unreadable, or passphrase-protected with no agent available: preflight fails naming the host and the specific cause, with remediation guidance; mkind never falls back to prompting for a password.
- Host-to-host SSH fails after the per-cluster key is distributed but before nodes are provisioned: creation aborts, the distributed key material is removed from every host it reached, and the user is told which host pair failed.
- A previous cluster's per-cluster key material is found on a host during creation (e.g., a delete that could not complete): creation reports the stale credential and its cluster of origin rather than silently reusing or overwriting it.
- Two hosts are each reachable from the workstation but cannot reach each other (over SSH or over the k8s-node address ranges): preflight fails naming the host pair and the direction that failed, before any resources are created.
- A host is missing its container runtime, lacks required privileges, or has packet forwarding disabled: preflight checks report the specific deficiency per host with remediation guidance.
- The config places control-plane nodes on more than one host: validation rejects it, naming the conflicting hosts.
- The config requests an even number of control-plane nodes: validation rejects it, stating the allowed counts.
- A node fails to join the cluster mid-creation: the tool reports actionable diagnostics for that k8s-node (k8s-node address, join output, connectivity status) and continues bootstrapping the remaining k8s-nodes where possible, ending with a clear summary of healthy vs. failed k8s-nodes.
- The config requests more per-host k8s-node address space than the configured pool allows (pool exhaustion): validation fails with a message stating the limit.
- A host named in the config already participates in another mkind cluster: creation is refused, naming the occupying cluster and the host, before any resources are created. (One cluster per host is an MVP limit; see Assumptions.)
- A cluster's requested ranges would overlap pre-existing host networking: creation is refused with the conflicting range identified.
- A cluster with the requested name already exists and the config is unchanged: creation attempts best-effort convergence, reporting per k8s-node what was skipped and what was created (FR-035). It never duplicates k8s-nodes or reallocates addresses.
- A cluster with the requested name already exists but the config differs: creation is refused, identifying what changed; the user is told to delete or rename (FR-036).
- Best-effort convergence cannot complete (a surviving node is unhealthy in a way re-creation cannot fix): creation stops and directs the user to delete and recreate, rather than leaving the cluster further degraded.
- The kubeconfig already holds a context with the name this cluster would claim: the conflict is reported rather than silently overwriting an unrelated context; the merge is refused or the user is directed to an explicit target file.
- The user's kubeconfig is unwritable, malformed, or absent at merge time: the cluster itself is left running and healthy, the failure is reported as a credential-delivery failure only, and the user is told how to re-export.
- A host reboots while cluster creation is in progress: the operation fails for that host's nodes with a clear indication of partial state, and the delete command can clean it up.
- A host's own network management reclaims or removes the tool's persisted routes after a reboot: the cluster reports the affected k8s-nodes as unreachable with the missing routes identified, rather than appearing healthy.
- Deletion is invoked when some hosts are unreachable: resources on reachable hosts are removed, and the user receives a per-host report of what could not be cleaned and how to retry.
- The user's workstation loses connectivity to a host mid-creation: the operation fails with a clear indication of partial state and how to clean it up (delete command works on partial clusters).
- The same create command is run twice concurrently for the same cluster name: the second invocation is refused or safely queued, never producing interleaved half-clusters.

## Requirements *(mandatory)*

### Network Requirements

These are environmental prerequisites the tool verifies during preflight (FR-005, FR-006) rather than provisions:

- **NR-001**: The user's workstation MUST reach every configured host over **passwordless, key-based SSH** using the identity named in the config or the user's SSH agent. Establishing this access is an operator prerequisite completed *before* mkind is run; mkind verifies it, never configures it.
- **NR-002**: The network path and SSH service configuration MUST permit direct host-to-host SSH connections between every pair of configured hosts. Authentication for those connections is supplied by mkind's own per-cluster keypair (FR-029), not by operator-configured trust.
- **NR-003**: Hosts MUST have direct, routable IP connectivity to one another for the k8s-node address ranges, with no address translation or stateful filtering rewriting or dropping traffic between those ranges. Standard Kubernetes control-plane, kubelet, and pod traffic flows over these ranges.
- **NR-004**: The network path between hosts MUST honor the static routes the tool installs for peer hosts' k8s-node subnets — hosts are on the same L2 segment, or on an L3 network whose intervening equipment forwards those subnets.
- **NR-005**: The user's workstation MUST reach the cluster's access endpoint on the control-plane host, so the delivered credentials work from the workstation.
- **NR-006**: The path between hosts MUST have a consistent MTU; the pod network MTU must not exceed the smallest MTU on that path.
- **NR-007**: Hosts MUST permit packet forwarding and route programming, or permit the tool to enable them.

### Functional Requirements

**Configuration and validation**

- **FR-001**: The system MUST accept a declarative, versioned cluster configuration file that defines the cluster name, Kubernetes version, participating hosts (address, login user, and the SSH identity used to reach that host), the k8s-nodes placed on each host, each k8s-node's role (control-plane or worker), and the address ranges used for nodes, pods, and services.
- **FR-002**: The system MUST validate the configuration before contacting any host and reject invalid configs with errors that identify the offending field and reason.
- **FR-003**: The system MUST reject any config that places control-plane nodes on more than one host, identifying the conflicting placements. All control-plane k8s-nodes of a cluster belong to exactly one host.
- **FR-004**: The system MUST reject any config with an even number of control-plane nodes, stating the allowed counts (1, 3, 5, …).

**Preflight**

- **FR-005**: The system MUST verify per-host prerequisites (passwordless key-based SSH authentication from the workstation, container runtime availability, required privileges, packet forwarding) before creating resources, and report deficiencies per host.
- **FR-006**: The system MUST verify host-to-host prerequisites before creating resources — network-path reachability to the SSH port between every pair of configured hosts, and IP reachability for the k8s-node address ranges (NR-002, NR-003) — reporting failures per host pair and direction. Because host-to-host *authentication* depends on the per-cluster keypair (FR-029), which does not exist before creation begins, preflight verifies only the path; authenticated host-to-host SSH MUST be verified immediately after key distribution and before any k8s-node is provisioned, so a failure there still leaves no cluster resources behind.

- **FR-031**: A host MUST belong to at most one mkind cluster at a time. Preflight MUST detect a host that already participates in a *different* cluster and refuse creation, naming the occupying cluster and the host, before any resources are created. A host already belonging to the cluster being created is not a conflict — that is the re-run path (FR-035). This behavior MUST NOT be left undefined. (Multiple clusters per host is deliberate future work — see `docs/PROJECT_GOAL_LONGTERM.md`.)

**Host access and credentials**

> Requirement IDs are stable: new requirements are appended with the next free number rather than renumbering, since tests and tasks will reference them.

- **FR-027**: The system MUST authenticate to hosts using key-based SSH only. It MUST NOT prompt for, accept, or store passwords; a config supplying a password MUST be rejected during validation.
- **FR-028**: The user's own SSH identity MUST be used only for workstation-to-host connections and MUST NOT be copied to, or stored on, any host.
- **FR-029**: At cluster creation the system MUST generate a dedicated per-cluster SSH keypair for host-to-host access, and install it on each participating host with an `authorized_keys` entry restricted to that cluster's use and private-key permissions readable only by the owning user. It MUST NOT reuse the user's key material for this purpose.
- **FR-030**: The per-cluster keypair MUST persist across host reboots so that host-to-host access works with no user involvement (supporting FR-019), and MUST be removed from every participating host on cluster deletion (extending FR-021), leaving no usable credential behind.

**Creation**

- **FR-007**: Users MUST be able to create a cluster with a single command, using standard remote shell access (SSH) to the hosts and no software manually pre-installed on them beyond the documented host prerequisites.
- **FR-008**: The system MUST support clusters on a single host and clusters spanning two or more hosts, through the same config schema and the same commands.
- **FR-009**: The system MUST provision k8s-nodes on each configured remote host according to the k8s-node placement in the config.
- **FR-010**: The system MUST allocate each host a unique, non-overlapping k8s-node address range from the configured pool, deterministically — the same config always yields the same allocation.
- **FR-011**: The system MUST make k8s-node addresses reachable across all participating hosts so that inter-node, pod-to-pod, and node-to-control-plane communication works without any change to standard cluster networking components.
- **FR-012**: The system MUST bootstrap a single-control-plane cluster (one control-plane node, workers on the same or other hosts) to a state where all k8s-nodes report Ready.
- **FR-013**: The system MUST bootstrap a multi-member control plane of 3 or 5 control-plane nodes, all co-located on one host, coordinated as one cluster behind a single stable access endpoint.
- **FR-014**: The system MUST join worker nodes running on any configured host to the cluster, regardless of which host holds the control plane.
- **FR-015**: The system MUST deliver working cluster access credentials to the user's workstation upon successful creation, usable with standard Kubernetes tooling. The credentials MUST be taken from the administrator config generated on a control-plane node during bootstrap; the system MUST NOT mint its own.
- **FR-033**: The delivered kubeconfig's server address MUST be rewritten from the node-local address recorded during bootstrap to the cluster's stable endpoint as reached from the workstation (NR-005). It MUST point at the cluster's single stable endpoint rather than at any individual control-plane node, so that access survives the restart or loss of one control-plane member (FR-013, FR-019).
- **FR-034**: The system MUST merge the delivered kubeconfig into the user's default kubeconfig as a context named for the cluster and set it as the current context, MUST accept an explicit target file to write instead, and MUST provide a verb that regenerates and re-exports the kubeconfig for an existing cluster without recreating it.
- **FR-035**: Re-running creation against an existing cluster with an unchanged config MUST be safe: it MUST NOT duplicate nodes, reallocate addresses, or corrupt a healthy cluster. It SHOULD converge the cluster on a best-effort basis — leaving healthy k8s-nodes untouched and provisioning only what is missing — and MUST report per node what it skipped and what it created. Where it cannot converge, it MUST fail with a clear instruction to delete and recreate rather than leave the cluster in a worse state. Best-effort convergence is explicitly not a guarantee; `delete` followed by `create` is the supported recovery path (SC-012).
- **FR-036**: Re-running creation with a config that differs from the existing cluster's topology MUST be refused, identifying what changed, rather than attempted as an in-place migration.

**Reboot resilience**

- **FR-016**: Node containers MUST be configured to start automatically when their host's container runtime comes up after a reboot, without any user command.
- **FR-017**: All host network configuration the tool creates — per-host k8s-node subnet, static routes to peer hosts' k8s-node subnets, and packet forwarding — MUST survive a host reboot, either by persisting or by being re-established automatically at boot.
- **FR-018**: Node addresses and k8s-node identities MUST remain stable across container restarts and host reboots, so that routes and cluster membership stay valid without re-bootstrapping.
- **FR-019**: After any subset of a cluster's hosts — up to and including all of them — reboots and comes back online, the cluster MUST return to all-k8s-nodes-Ready with no user commands, and the credentials issued at creation MUST continue to work unchanged.

**Lifecycle**

- **FR-020**: Users MUST be able to list existing clusters and list a cluster's k8s-nodes with role, status, and hosting machine. In this phase the commands take the cluster config and inspect exactly the hosts it names; discovering clusters without being given a config is deliberate future work (see `docs/PROJECT_GOAL_LONGTERM.md`).
- **FR-032**: Every k8s-node container the system creates MUST carry labels identifying its cluster and its role, so that listing (FR-020), occupancy detection (FR-031), and cleanup (FR-021) all work by querying a host's container runtime rather than from any stored record. Labels are the on-host source of truth.
- **FR-021**: Users MUST be able to delete a cluster by name, removing all tool-created resources from every participating host — k8s-node containers, network configuration, routing entries, any reboot-persistence settings installed under FR-016 and FR-017, and the per-cluster SSH keypair and its `authorized_keys` entries installed under FR-029 — such that nothing from the deleted cluster reappears after a host reboot and no usable credential is left behind. Deletion MUST also remove the cluster's context and its associated user and cluster entries from the kubeconfig it was merged into (FR-034), leaving other contexts untouched.
- **FR-022**: The system MUST support preloading required k8s-node images onto all configured hosts ahead of cluster creation, and MUST use locally present images without re-downloading.

**Behavior**

- **FR-023**: On any k8s-node-level failure during creation, the system MUST emit actionable diagnostics (k8s-node identity, host, failure output, connectivity status), continue with unaffected k8s-nodes where possible, and finish with an accurate per-k8s-node outcome summary.
- **FR-024**: All commands MUST support both human-readable and machine-readable (JSON) output; errors MUST go to the error stream and data to the standard output stream.
- **FR-025**: The system MUST NOT require any external services beyond the configured hosts for cluster creation; all cluster state MUST be recoverable from the running k8s-node containers and the config file.
- **FR-026**: Cluster and command behavior MUST follow Kubernetes ecosystem conventions: typed and versioned config objects, verb-noun command structure, and config files (not flags) for structured values.

### Key Entities

- **Cluster**: A named Kubernetes cluster spanning one or more hosts; owns its k8s-nodes, its address-range allocations, and its access endpoint. Identified uniquely by name.
- **Cluster Config**: The user-authored, versioned declaration of a desired cluster — name, Kubernetes version, network address pools (nodes, pods, services), and the list of hosts with their k8s-node placements. The single source of truth for creation.
- **Host**: A remote Linux machine participating in a cluster; reachable by address with a login user and an SSH identity, from the workstation and from every peer host; receives a unique k8s-node address range; runs zero or more k8s-nodes. Exactly one host of a cluster holds its control-plane nodes.
- **Cluster Access Key**: A per-cluster SSH keypair generated at creation and installed on every participating host solely to authenticate host-to-host connections. Distinct from the user's own identity, persists across reboots, and is destroyed with the cluster.
- **K8s-Node**: A containerized Kubernetes node running on a host, with a role (control-plane or worker), a stable unique address from its host's range, and a lifecycle bound to its cluster — including automatic restart after its host reboots.
- **K8s-Node Address Allocation**: The deterministic assignment of a per-host address range from the cluster's k8s-node address pool; must never overlap between hosts of the same cluster or with other allocations on the same host, and must be reproduced identically after a reboot.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user with one prepared host can go from a written config file to a fully Ready single-host cluster with a single command in under 5 minutes (with images already present).
- **SC-002**: A user with two prepared hosts can go from a written config file to a fully Ready two-host cluster with a single command in under 10 minutes (with images already present on hosts).
- **SC-003**: 100% of pods deployed without placement constraints on a two-host cluster can exchange traffic with pods on the other host, verified by bidirectional connectivity checks.
- **SC-004**: A cluster with three control-plane nodes on one host and workers on at least one other host reaches all-k8s-nodes-Ready and remains manageable through a single endpoint; the same holds with five control-plane nodes.
- **SC-005**: After rebooting any single participating host, the cluster returns to all-k8s-nodes-Ready within 10 minutes of that host coming up, with zero commands run by the user.
- **SC-006**: After rebooting every participating host simultaneously, the cluster returns to all-k8s-nodes-Ready within 15 minutes of the last host coming up, with zero commands run by the user, and the credentials issued at creation still work.
- **SC-007**: After deleting a cluster, zero tool-created resources remain on any participating host — k8s-node containers, routes, persistence settings, and the per-cluster SSH key material and its `authorized_keys` entries — including after rebooting each host, and recreating a cluster with the same name succeeds on the first attempt without manual cleanup.
- **SC-008**: Creating the same cluster config twice (after deletion) produces identical k8s-node addressing and topology both times; so does recreating the same cluster's state after a reboot.
- **SC-009**: With images preloaded, cluster creation completes successfully with external image sources unreachable, and creation time is reduced compared to the non-preloaded path.
- **SC-010**: When a k8s-node join fails, the user can identify the failing node, its host, and the failure reason from command output alone — without logging in to any host.
- **SC-011**: An invalid or unsatisfiable config never results in partial resources on any host: 100% of validation and preflight failures occur before remote changes begin.
- **SC-012**: After any partially failed creation, `delete` followed by `create` yields a fully Ready cluster on the first attempt, 100% of the time, with no manual host cleanup. Re-running `create` alone recovers some partial failures but carries no such guarantee.

## Assumptions

- Target users are Kubernetes practitioners (developers, testers, platform engineers) comfortable with declarative config files and standard Kubernetes client tooling.
- Participating hosts are Linux machines with a container runtime installed, remote shell (SSH) access with sufficient privileges, and mutual network reachability as specified in Network Requirements (NR-001 – NR-007).
- **Passwordless, key-based SSH from the workstation to every host is set up by the operator before mkind is run.** mkind verifies this and fails fast when it is absent; it never configures SSH access, never prompts for a password, and never accepts password credentials in the config. Host-to-host authentication is the one exception mkind manages itself, via the per-cluster keypair (FR-029) — chosen because reboot recovery (FR-019) must work with the workstation powered off, which rules out agent forwarding, and because the operator's own key must never be copied onto a host.
- **All control-plane k8s-nodes of a cluster live on a single host.** This is an MVP constraint, chosen so that datastore quorum traffic never crosses a host boundary and cross-host reachability is only needed for worker-to-control-plane and pod-to-pod traffic. Its consequence is explicit: losing the control-plane host takes the cluster's control plane offline until that host returns. The multi-member control plane in Story 4 provides production-like topology and control-plane process redundancy — not host-level fault tolerance.
- Control-plane node counts are restricted to odd numbers so the datastore quorum is well-defined; even counts are rejected rather than silently degraded.
- **Host reboots are in scope and expected.** Recovery from a reboot is automatic (Story 5). What remains out of scope for this MVP phase: repairing or replacing a permanently failed node or host, cluster upgrades, host autoscaling, and non-Linux hosts.
- Cluster creation requires access to k8s-node images either via network download or prior preloading (Story 7); fully air-gapped operation is supported only through preloading.
- Alternative cross-host network topologies (tunnels, overlays) are out of scope; hosts with direct mutual reachability are the supported environment, consistent with the project constitution's simplicity principle.
- Cluster state observation (`list` commands) reflects state discoverable from the hosts and running containers, keyed off container labels (FR-032); no separate state database is maintained. In the MVP the set of hosts to inspect comes from the cluster config the user supplies — mkind keeps no workstation-side inventory, so it cannot find a cluster it is not pointed at. Reboot recovery therefore relies on host-local persistence, not on a coordinator running on the workstation — the workstation may be off while hosts reboot.
- **One mkind cluster per host in the MVP.** A host cannot participate in two clusters simultaneously; creation refuses rather than leaving the outcome undefined (FR-031). Supporting several clusters across the same subset of hosts — the CI-parallelism case — is wanted and is recorded as future work in `docs/PROJECT_GOAL_LONGTERM.md`.
- A single user operates on a given cluster at a time; concurrent multi-user coordination on the same hosts is out of scope.
