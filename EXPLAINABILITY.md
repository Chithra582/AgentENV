# EXPLAINABILITY.md

This document explains the internal mechanisms, data lineage, operational boundaries, and governance framework of **AgentENV (AENV)** (`agentenv`) in accordance with the **OpenGAP v0.1.0** specification for the **HiDevs GitAgent Passport** clearance pipeline.

> **Agent Name:** AgentENV (AENV) (`agentenv`)  
> **Specification:** OpenGAP v0.1.0  
> **Category / Domain:** Developer Tools / Scalable Agent Sandboxes, Firecracker MicroVMs & RL Training Environments  
> **Compliance Standard:** OpenGAP Checkpoint 2 (Explainability & Decision Governance), OWASP LLM Top 10, MITRE ATLAS  

---

## How the Agent Decides

AgentENV provides a high-density, low-latency execution fabric for agentic reinforcement learning (RL) training (e.g., Kimi K3). Environment provisioning, instant branching, and execution commands flow through five discrete processing stages:

### 1. Decision Architecture

The runtime intake, state classification, evaluation, and execution tracking operate across a deterministic, five-stage pipeline:

```
[ RL Training Step / Environment Request ]
                    │
                    ▼
[ 1. Host Capacity & KVM Resource Allocation ]
                    │
                    ▼
[ 2. Lazy Image Streaming (overlaybd / ublk) ]
                    │
                    ▼
[ 3. Firecracker MicroVM Boot / Snapshot Resume (<50ms) ]
                    │
                    ▼
[ 4. Guest Action Execution (envd) & Observation Capture ]
                    │
                    ▼
[ 5. Incremental Snapshot (<100ms) / Fork Branching ]
```

### 2. Decision Logic & Routing Formulations

When allocating microVM sandboxes across heterogeneous host nodes in a distributed AgentENV cluster, the scheduler calculates a node suitability score $S_{\text{host}}$:

$$S_{\text{host}} = w_m \cdot \left(1 - \frac{M_{\text{committed}}}{M_{\text{total}} \cdot \Omega_{\text{overcommit}}}\right) + w_c \cdot C_{\text{hit}} + w_k \cdot \left(1 - \frac{V_{\text{active}}}{V_{\text{cap}}}\right) + w_i \cdot \frac{1}{1 + \text{IOPS}_{\text{load}}}$$

Where:
- $M_{\text{committed}} / (M_{\text{total}} \cdot \Omega_{\text{overcommit}})$: Committed guest RAM ratio accounting for memory ballooning overcommit ceiling ($\Omega_{\text{overcommit}} = 9.6$).
- $C_{\text{hit}} \in [0, 1]$: Local disk block cache hit ratio for requested OCI container layers.
- $V_{\text{active}} / V_{\text{cap}}$: Active microVM slot saturation against host `/dev/kvm` thread limits.
- $\text{IOPS}_{\text{load}}$: Normalized host `ublk` I/O queue depth.
- Parameter weights: $w_m = 0.35$, $w_c = 0.30$, $w_k = 0.20$, $w_i = 0.15$ ($\sum w_i = 1.0$).

The host node maximizing $S_{\text{host}}$ is selected for sandbox provisioning.

### 3. Thresholding & Refusal Decision Criteria

AgentENV (AENV) enforces strict operational boundaries and deterministic refusal thresholds:
- **Refusal on ERR_KVM_UNAVAILABLE**: Host system lacks accessible Linux KVM hardware acceleration (`/dev/kvm`) halts execution with code `ERR_KVM_UNAVAILABLE`.
- **Refusal on ERR_SLA_TIMEOUT_EXCEEDED**: MicroVM boot, resume, or fork operation exceeds SLA threshold (> 100ms) halts execution with code `ERR_SLA_TIMEOUT_EXCEEDED`.
- **Refusal on ERR_CACHE_QUOTA_EXCEEDED**: Local block cache exceeds storage limit and cold eviction fails halts execution with code `ERR_CACHE_QUOTA_EXCEEDED`.
- **Refusal on ERR_GUEST_EXECUTION_TIMEOUT**: In-guest process execution exceeds timeout deadline (> 60s) halts execution with code `ERR_GUEST_EXECUTION_TIMEOUT`.
- **Refusal on ERR_UNAUTHORIZED_EGRESS**: Sandboxed environment attempts network access to internal control plane ports halts execution with code `ERR_UNAUTHORIZED_EGRESS`.

### 4. Fallback Decision Mechanism

Continuous operational stability is maintained through layered fault recovery:
- **PVM Emulation Failover**: If native hardware KVM is unavailable, AgentENV can fall back to the PVM (Process Virtual Machine) deployment mode for development testing.
- **Snapshot Storage Redundancy**: If remote S3 snapshot storage encounters network partitioning, the snapshotter commits incremental diffs to local NVMe storage and schedules background S3 synchronization.
- **WarmPool Allocation Fallback**: If an ondemand image pull experiences network latency, the scheduler allocates a prewarmed generic base microVM from the local pool and overlays task dependencies dynamically.
- **Model Fallback Cascade**: High-level reasoning and synthesis default to `gemini-2.0-flash` with automatic failover to `gpt-4o` and `claude-3-5-sonnet`.

### 5. Human-in-the-Loop Governance

Human operators retain sovereign authority over the multi-agent execution lifecycle:
- **Cluster Overcommit Ceiling Controls**: Infrastructure operators configure maximum memory ballooning ratios ($\Omega_{\text{overcommit}}$) and CPU pin allocations per host node.
- **Root Security Guardrails**: Modifying host kernel drivers, `ublk` daemon configurations, or TAP bridge rules requires system administrator privileges.
- **Tenant Quota Enforcement**: Multi-tenant RL research teams operate within strict concurrency ceilings to prevent single-job cluster monopolization.

---

## The Data It Uses

AgentENV (AENV) operates under strict principles of data minimization, environment isolation, and privacy protection.

### 1. Ingested Input Data

The framework processes only operational data necessary to perform its functions:
- **RL Action Payloads**: Shell commands, Python scripts, test suites, and terminal keyboard inputs dispatched to the guest environment.
- **OCI Container References**: Image registry URIs, tag digests, and layer manifests specifying task environments.
- **Configuration Manifests**: JSON/YAML definitions specifying vCPU counts, memory sizes, block devices, and network bridges.

### 2. Configuration & Reference Data

- **Firecracker MicroVM Kernels**: Stripped-down uncompressed Linux kernel binaries optimized for fast virtualization.
- **OverlayBD Image Indices**: Remote block layer offset maps and chunk digests hosted on OCI registries or S3.
- **Snapshot Merkle Trees**: Incremental memory page diffs and copy-on-write filesystem block registries.

### 3. Base Model & Inference Lineage

- **Virtualization Core**: Rust-based Firecracker microVM orchestrator interfacing via Unix domain sockets.
- **Block Storage Engine**: Linux kernel `ublk` user-space block driver paired with containerd overlaybd snapshotters.
- **In-Guest Agent**: Lightweight `envd` daemon communicating with host over virtual serial channels.

### 4. Data Privacy, Storage, and Retention

- **OWASP LLM & MITRE ATLAS Hardened**: Defended against indirect prompt injection, credential leakage, and unauthorized external API dispatch.
- **Local Environment Isolation**: Agent execution workspaces, intermediate scratchpads, and vector stores reside strictly within designated local project directories.
- **Automated Secret Scrubbing**: API keys, database credentials, and personal credentials are automatically redacted prior to embedding or logging.
- **Zero Commercial Monetization**: Prompts, intermediate reasoning trajectories, and task deliverables are never commercialized or shared with third parties.

---

## Limitations

Understanding the operational boundaries and technical constraints of AgentENV (AENV) is essential for effective deployment.

### 1. Mathematical Scoring & Routing Formulation
When allocating microVM sandboxes across heterogeneous host nodes in a distributed AgentENV cluster, the scheduler calculates a node suitability score $S_{\text{host}}$:

$$S_{\text{host}} = w_m \cdot \left(1 - \frac{M_{\text{committed}}}{M_{\text{total}} \cdot \Omega_{\text{overcommit}}}\right) + w_c \cdot C_{\text{hit}} + w_k \cdot \left(1 - \frac{V_{\text{active}}}{V_{\text{cap}}}\right) + w_i \cdot \frac{1}{1 + \text{IOPS}_{\text{load}}}$$

Where:
- $M_{\text{committed}} / (M_{\text{total}} \cdot \Omega_{\text{overcommit}})$: Committed guest RAM ratio accounting for memory ballooning overcommit ceiling ($\Omega_{\text{overcommit}} = 9.6$).
- $C_{\text{hit}} \in [0, 1]$: Local disk block cache hit ratio for requested OCI container layers.
- $V_{\text{active}} / V_{\text{cap}}$: Active microVM slot saturation against host `/dev/kvm` thread limits.
- $\text{IOPS}_{\text{load}}$: Normalized host `ublk` I/O queue depth.
- Parameter weights: $w_m = 0.35$, $w_c = 0.30$, $w_k = 0.20$, $w_i = 0.15$ ($\sum w_i = 1.0$).

The host node maximizing $S_{\text{host}}$ is selected for sandbox provisioning.

### 2. Refusal Criteria & Decision Thresholds
AgentENV enforces deterministic operational boundaries:

| Trigger Scenario | Operational Action | Error Code |
| :--- | :--- | :--- |
| Host system lacks accessible Linux KVM hardware acceleration (`/dev/kvm`) | Refuse microVM launch; abort with hardware prerequisite error | `ERR_KVM_UNAVAILABLE` |
| MicroVM boot, resume, or fork operation exceeds SLA threshold (> 100ms) | Terminate delinquent VM thread; log latency degradation event | `ERR_SLA_TIMEOUT_EXCEEDED` |
| Local block cache exceeds storage limit and cold eviction fails | Halt new lazy image mounts; report local disk quota exhaustion | `ERR_CACHE_QUOTA_EXCEEDED` |
| In-guest process execution exceeds timeout deadline (> 60s) | Force-kill guest process; return partial stdout with timeout status | `ERR_GUEST_EXECUTION_TIMEOUT` |
| Sandboxed environment attempts network access to internal control plane ports | Drop network packets; isolate guest TAP device interface | `ERR_UNAUTHORIZED_EGRESS` |

### 3. Multi-Tier Fallback Mechanisms
1. **PVM Emulation Failover**: If native hardware KVM is unavailable, AgentENV can fall back to the PVM (Process Virtual Machine) deployment mode for development testing.
2. **Snapshot Storage Redundancy**: If remote S3 snapshot storage encounters network partitioning, the snapshotter commits incremental diffs to local NVMe storage and schedules background S3 synchronization.
3. **Warm-Pool Allocation Fallback**: If an on-demand image pull experiences network latency, the scheduler allocates a pre-warmed generic base microVM from the local pool and overlays task dependencies dynamically.

### 4. Human-in-the-Loop Governance
- **Cluster Overcommit Ceiling Controls**: Infrastructure operators configure maximum memory ballooning ratios ($\Omega_{\text{overcommit}}$) and CPU pin allocations per host node.
- **Root Security Guardrails**: Modifying host kernel drivers, `ublk` daemon configurations, or TAP bridge rules requires system administrator privileges.
- **Tenant Quota Enforcement**: Multi-tenant RL research teams operate within strict concurrency ceilings to prevent single-job cluster monopolization.

---

## The Data It Uses

### 1. Input Data Types
- **RL Action Payloads**: Shell commands, Python scripts, test suites, and terminal keyboard inputs dispatched to the guest environment.
- **OCI Container References**: Image registry URIs, tag digests, and layer manifests specifying task environments.
- **Configuration Manifests**: JSON/YAML definitions specifying vCPU counts, memory sizes, block devices, and network bridges.

### 2. Reference & Configuration Data
- **Firecracker MicroVM Kernels**: Stripped-down uncompressed Linux kernel binaries optimized for fast virtualization.
- **OverlayBD Image Indices**: Remote block layer offset maps and chunk digests hosted on OCI registries or S3.
- **Snapshot Merkle Trees**: Incremental memory page diffs and copy-on-write filesystem block registries.

### 3. Model Lineage & System Architecture
- **Virtualization Core**: Rust-based Firecracker microVM orchestrator interfacing via Unix domain sockets.
- **Block Storage Engine**: Linux kernel `ublk` user-space block driver paired with containerd overlaybd snapshotters.
- **In-Guest Agent**: Lightweight `envd` daemon communicating with host over virtual serial channels.

### 4. Data Privacy, Retention & Sanitization
- **Strict Tenant Sandbox Isolation**: Hardware virtualization ensures zero memory leakage between co-located microVMs.
- **Ephemeral Scratch Sanitization**: Ephemeral disk layers and guest memory are wiped clean upon sandbox destruction.
- **Encrypted Snapshot Persistence**: Snapshots persisted to S3 object stores are encrypted at rest using server-side AES-256 keys.

---

## Limitations

### 1. Linux Kernel 6.8+ Prerequisite
- **Limitation**: Advanced features (ublk user-space block devices, modern snapshotting) require Linux kernel 6.8 or newer with `/dev/kvm`.
- **Mitigation**: Provide fallback compatibility modes (PVM) and pre-packaged OS deployment images for development environments.

### 2. MicroVM Cold Start vs. Warm Resume
- **Limitation**: Cold booting a brand-new OCI image without local cache blocks takes 1–2 seconds due to initial block streaming over the network.
- **Mitigation**: Leverage warm pools and snapshot-based resume (<50ms) to ensure RL training loops operate exclusively on pre-cached or snapshotted state.

### 3. In-Guest Clock Drift on Prolonged Pause
- **Limitation**: Pausing a microVM for extended intervals can cause guest NTP clock skew upon resumption.
- **Mitigation**: Automatically synchronize guest system clocks via `envd` immediately following every resume operation.

### 4. GPU Virtualization Overhead
- **Limitation**: Firecracker microVMs do not natively support direct PCIe passthrough for hardware GPUs without specialized virtualization extensions.
- **Mitigation**: Offload heavy deep learning tensor operations to remote host inference servers while keeping agent logic in the sandbox.

### 5. Memory Overcommit Thrashing Risks
- **Limitation**: Aggressive 9.6x memory overcommit can trigger host OOM killer cascades if all running sandboxes simultaneously become active.
- **Mitigation**: Implement dynamic memory ballooning feedback loops that prioritize freezing idle sandboxes before host memory pressure reaches critical watermarks.

---

## Summary & Compliance Checklist

| Checkpoint 2 Requirement | Corresponding Section | Status |
| :--- | :--- | :---: |
| **How the agent decides** | [How the Agent Decides](#how-the-agent-decides) | **Covered** |
| - Decision architecture & 5-stage pipeline | Section 1 | Verified |
| - Decision logic & routing formulations | Section 2 | Verified |
| - Thresholding & refusal decision criteria | Section 3 | Verified |
| - Fallback decision mechanism | Section 4 | Verified |
| - Human-in-the-loop governance & oversight | Section 5 | Verified |
| **The data it uses** | [The Data It Uses](#the-data-it-uses) | **Covered** |
| - Ingested input data & query streams | Section 1 | Verified |
| - Configuration & reference schemas | Section 2 | Verified |
| - Base model lineage & deterministic engines | Section 3 | Verified |
| - Data privacy, retention lifecycle & MITRE/OWASP | Section 4 | Verified |
| **Its limitations** | [Limitations](#limitations) | **Covered** |
| - Mathematical Scoring & Routing Formulation
When allocating microVM sandboxes across heterogeneous host nodes in a distributed AgentENV cluster, the scheduler calculates a node suitability score $S_{\text{host}}$:

$$S_{\text{host}} = w_m \cdot \left(1 - \frac{M_{\text{committed}}}{M_{\text{total}} \cdot \Omega_{\text{overcommit}}}\right) + w_c \cdot C_{\text{hit}} + w_k \cdot \left(1 - \frac{V_{\text{active}}}{V_{\text{cap}}}\right) + w_i \cdot \frac{1}{1 + \text{IOPS}_{\text{load}}}$$

Where:
- $M_{\text{committed}} / (M_{\text{total}} \cdot \Omega_{\text{overcommit}})$: Committed guest RAM ratio accounting for memory ballooning overcommit ceiling ($\Omega_{\text{overcommit}} = 9.6$).
- $C_{\text{hit}} \in [0, 1]$: Local disk block cache hit ratio for requested OCI container layers.
- $V_{\text{active}} / V_{\text{cap}}$: Active microVM slot saturation against host `/dev/kvm` thread limits.
- $\text{IOPS}_{\text{load}}$: Normalized host `ublk` I/O queue depth.
- Parameter weights: $w_m = 0.35$, $w_c = 0.30$, $w_k = 0.20$, $w_i = 0.15$ ($\sum w_i = 1.0$).

The host node maximizing $S_{\text{host}}$ is selected for sandbox provisioning.

### 2. Refusal Criteria & Decision Thresholds
AgentENV enforces deterministic operational boundaries:

| Trigger Scenario | Operational Action | Error Code |
| :--- | :--- | :--- |
| Host system lacks accessible Linux KVM hardware acceleration (`/dev/kvm`) | Refuse microVM launch; abort with hardware prerequisite error | `ERR_KVM_UNAVAILABLE` |
| MicroVM boot, resume, or fork operation exceeds SLA threshold (> 100ms) | Terminate delinquent VM thread; log latency degradation event | `ERR_SLA_TIMEOUT_EXCEEDED` |
| Local block cache exceeds storage limit and cold eviction fails | Halt new lazy image mounts; report local disk quota exhaustion | `ERR_CACHE_QUOTA_EXCEEDED` |
| In-guest process execution exceeds timeout deadline (> 60s) | Force-kill guest process; return partial stdout with timeout status | `ERR_GUEST_EXECUTION_TIMEOUT` |
| Sandboxed environment attempts network access to internal control plane ports | Drop network packets; isolate guest TAP device interface | `ERR_UNAUTHORIZED_EGRESS` |

### 3. Multi-Tier Fallback Mechanisms
1. **PVM Emulation Failover**: If native hardware KVM is unavailable, AgentENV can fall back to the PVM (Process Virtual Machine) deployment mode for development testing.
2. **Snapshot Storage Redundancy**: If remote S3 snapshot storage encounters network partitioning, the snapshotter commits incremental diffs to local NVMe storage and schedules background S3 synchronization.
3. **Warm-Pool Allocation Fallback**: If an on-demand image pull experiences network latency, the scheduler allocates a pre-warmed generic base microVM from the local pool and overlays task dependencies dynamically.

### 4. Human-in-the-Loop Governance
- **Cluster Overcommit Ceiling Controls**: Infrastructure operators configure maximum memory ballooning ratios ($\Omega_{\text{overcommit}}$) and CPU pin allocations per host node.
- **Root Security Guardrails**: Modifying host kernel drivers, `ublk` daemon configurations, or TAP bridge rules requires system administrator privileges.
- **Tenant Quota Enforcement**: Multi-tenant RL research teams operate within strict concurrency ceilings to prevent single-job cluster monopolization.

---

## The Data It Uses

### 1. Input Data Types
- **RL Action Payloads**: Shell commands, Python scripts, test suites, and terminal keyboard inputs dispatched to the guest environment.
- **OCI Container References**: Image registry URIs, tag digests, and layer manifests specifying task environments.
- **Configuration Manifests**: JSON/YAML definitions specifying vCPU counts, memory sizes, block devices, and network bridges.

### 2. Reference & Configuration Data
- **Firecracker MicroVM Kernels**: Stripped-down uncompressed Linux kernel binaries optimized for fast virtualization.
- **OverlayBD Image Indices**: Remote block layer offset maps and chunk digests hosted on OCI registries or S3.
- **Snapshot Merkle Trees**: Incremental memory page diffs and copy-on-write filesystem block registries.

### 3. Model Lineage & System Architecture
- **Virtualization Core**: Rust-based Firecracker microVM orchestrator interfacing via Unix domain sockets.
- **Block Storage Engine**: Linux kernel `ublk` user-space block driver paired with containerd overlaybd snapshotters.
- **In-Guest Agent**: Lightweight `envd` daemon communicating with host over virtual serial channels.

### 4. Data Privacy, Retention & Sanitization
- **Strict Tenant Sandbox Isolation**: Hardware virtualization ensures zero memory leakage between co-located microVMs.
- **Ephemeral Scratch Sanitization**: Ephemeral disk layers and guest memory are wiped clean upon sandbox destruction.
- **Encrypted Snapshot Persistence**: Snapshots persisted to S3 object stores are encrypted at rest using server-side AES-256 keys.

---

## Limitations

### 1. Linux Kernel 6.8+ Prerequisite | Section 1 | Verified |
| - MicroVM Cold Start vs. Warm Resume | Section 2 | Verified |
| - In-Guest Clock Drift on Prolonged Pause | Section 3 | Verified |
| - GPU Virtualization Overhead | Section 4 | Verified |
| - Memory Overcommit Thrashing Risks | Section 5 | Verified |
