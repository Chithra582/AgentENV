# DUTIES — AgentENV (AENV)

## Core Agent Duties

### 1. High-Density MicroVM Lifecycle Management
- Provision, configure, and launch Firecracker microVM instances with custom vCPU and memory profiles.
- Pause idle environments to release CPU resources and resume paused instances in under 50 ms.
- Manage memory ballooning to reclaim unused guest RAM, enabling up to 9.6x memory overcommit density.

### 2. Instantaneous Snapshot & Branching Operations
- Take incremental memory snapshots and copy-on-write filesystem snapshots in under 100 ms.
- Fork running environments into multiple independent child sandboxes for tree search (MCTS) and parallel RL rollouts.
- Persist snapshot archives to S3-compatible object storage or shared distributed storage systems.

### 3. On-Demand Image Streaming & Block Caching
- Stream OCI container images lazily over the network via overlaybd without full image pre-pulling.
- Manage high-performance user-space block devices via `ublk` drivers and shared page cache.
- Maintain bounded local disk caches, retaining hot filesystem blocks and evicting cold data automatically.

### 4. Guest Environment Command & Observation Control
- Interface with the in-guest `envd` daemon over virtual serial or vsock communication channels.
- Dispatch shell scripts, execute benchmark binaries, and capture stdout, stderr, and exit codes.
- Manage bi-directional file transfers between the host orchestrator and guest filesystem.

### 5. Cluster Health & Resource Telemetry
- Monitor host `/dev/kvm` capacity, memory pressure, active microVM counts, and block cache hit rates.
- Expose Prometheus metrics and OpenTelemetry traces for real-time observability of RL environment clusters.
- Coordinate warm pool pre-allocation to satisfy bursty RL environment allocation spikes.
