# RULES — AgentENV (AENV)

## Operational Rules & Guardrails
1. **Strict Sub-100ms Latency Deadlines**: Environment lifecycle operations (pause, resume, fork, snapshot) must adhere to strict SLA targets (<50ms boot/resume, <100ms pause/fork); operations exceeding deadlines must be flagged for performance degradation.
2. **KVM Hardware Virtualization Enforcement**: MicroVMs must run with hardware-assisted virtualization (`/dev/kvm`); running unisolated processes or falling back to non-isolated containers without explicit configuration is prohibited.
3. **Bounded Local Cache Eviction**: Local block storage caches must strictly honor configured size caps, evicting cold image blocks using LRU policies to prevent host disk exhaustion across millions of loaded images.
4. **Isolated Guest Networking**: Each microVM sandbox must reside in a dedicated network namespace with private TAP devices; direct cross-guest network communication without authorized bridging is blocked.
5. **Execution Command Timeout Enforcement**: Commands executed inside guest microVMs via `envd` must specify an execution timeout (default: 60s); runaway processes must be killed automatically to prevent thread starvation.
6. **Encrypted Transport Requirement for Production**: In production clusters, AgentENV API communication must terminate at a secure TLS reverse proxy or VPN; plaintext transmission of API keys over untrusted networks is forbidden.
7. **Snapshot Integrity Verification**: All incremental memory and filesystem snapshots persisted to remote S3 storage must include cryptographic checksums to guarantee uncorrupted restoration.
