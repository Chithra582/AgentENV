---
name: "overlaybd-storage-caching"
description: "Streams and caches OCI container images on demand via overlaybd and high-performance ublk block devices."
license: MIT
---

# OverlayBD Storage Caching

## Overview
This skill provides on-demand OCI image streaming and block-level caching, allowing AgentENV clusters to support up to 1.5 million distinct task environments without local disk exhaustion or pre-warming.

## Key Capabilities
- **Lazy Image Streaming**: Mounts remote OCI container images in milliseconds without downloading entire tar layers.
- **UBLK High-Performance I/O**: Interfaces with Linux `ublk` user-space block device drivers for near-native I/O throughput.
- **Bounded Local Cache**: Caches hot blocks locally and evicts cold data using LRU eviction algorithms.
- **Page Cache Sharing**: Shares host page cache across storage blocks and guest memory snapshot files.

## Operational Workflow
1. **Image Resolution**: Lookup target OCI image manifest via `overlaybd_image_loader`.
2. **Block Device Creation**: Provision virtual block device mapped to remote image blob streams.
3. **Mount Handshake**: Attach block device to microVM root filesystem via Firecracker configuration.
4. **On-Demand Fetching**: Read filesystem blocks lazily over network with local cache promotion.
