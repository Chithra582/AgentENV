---
name: "agentic-rl-environment-control"
description: "Provides programmatic agent environment interfaces, command execution, and state observation for RL training."
license: MIT
---

# Agentic RL Environment Control

## Overview
This skill provides programmatic control and observation channels for autonomous reinforcement learning agents (such as Kimi K3), allowing agents to execute actions and receive rewards inside isolated sandboxes.

## Key Capabilities
- **In-Guest Command Dispatch**: Transmits shell commands, code scripts, and test invocations via `guest_command_executor`.
- **Output & Observation Capture**: Captures stdout, stderr, process return codes, and file diffs as agent observations.
- **Environment Reset & Rollback**: Reverts sandbox state to predefined checkpoints instantly for the next RL episode.
- **Gym-Style Interface**: Exposes step, reset, and observe APIs for integration into standard RL training loops.

## Operational Workflow
1. **Session Binding**: Connect agent session to active microVM instance.
2. **Action Dispatch**: Send action command (e.g., bash code execution, file edit) to in-guest `envd` daemon.
3. **Observation Harvesting**: Collect stdout streams, return code, and filesystem modifications.
4. **Step Evaluation**: Calculate reward and determine episode termination or transition to next step.
