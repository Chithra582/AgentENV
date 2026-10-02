---
name: "microvm-sandbox-orchestration"
description: "Boots, pauses, resumes, and terminates Firecracker microVM environments with sub-50ms latency."
license: MIT
---

# MicroVM Sandbox Orchestration

## Overview
This skill provisions, supervises, and reclaims lightweight Firecracker microVM execution environments, providing hardware-isolated Linux execution sandboxes at extreme cluster density.

## Key Capabilities
- **Sub-50ms MicroVM Startup**: Boots or resumes snapshot-backed guest VMs rapidly via Firecracker socket APIs.
- **Resource Elasticity**: Pauses idle guest sandboxes to yield CPU and memory back to the host system.
- **Memory Ballooning**: Dynamically inflates/deflates in-guest memory balloons to achieve 9.6x memory overcommit.
- **Hardware Isolation**: Enforces kernel-level KVM boundaries, preventing container escapes or cross-sandbox contamination.

## Operational Workflow
1. **VM Configuration**: Assemble guest vCPU, memory, kernel, and block device specs.
2. **Sandbox Provisioning**: Launch Firecracker process and configure resources via `microvm_lifecycle_manager`.
3. **Execution Monitoring**: Track guest health, CPU utilization, and memory pressure.
4. **Fast Teardown / Pause**: Pause or terminate guest VM upon task completion to reclaim host capacity.
