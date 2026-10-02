---
name: "snapshot-and-fork-management"
description: "Takes incremental memory and filesystem snapshots (<100ms) and forks running environments into parallel sandboxes."
license: MIT
---

# Snapshot and Fork Management

## Overview
This skill implements instantaneous state persistence and branching for active agent environments, enabling tree-search algorithms (e.g., MCTS) and parallel RL exploration trajectories.

## Key Capabilities
- **Sub-100ms Incremental Snapshots**: Captures guest memory pages and copy-on-write disk diffs with minimal pause overhead.
- **Instant Environment Forking**: Clones active running guest VMs into multiple independent child sandboxes via `sandbox_forker`.
- **S3 Persistence**: Streams compressed snapshot archives to S3-compatible object stores for cross-node replication.
- **Deterministic State Restoration**: Restores exact microVM execution states from snapshot files across heterogeneous hosts.

## Operational Workflow
1. **Snapshot Trigger**: Issue incremental snapshot command via `environment_snapshotter`.
2. **Memory & Disk Freeze**: Capture dirty memory pages and commit overlaybd block diffs.
3. **Branch Creation**: Fork new child environments referencing parent snapshot copy-on-write base.
4. **Independent Exploration**: Resume sibling sandboxes concurrently for parallel trajectory rollouts.
