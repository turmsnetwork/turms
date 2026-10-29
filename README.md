# TURMS

```
  ████████╗██╗   ██╗██████╗ ███╗   ███╗███████╗
  ╚══██╔══╝██║   ██║██╔══██╗████╗ ████║██╔════╝
     ██║   ██║   ██║██████╔╝██╔████╔██║███████╗
     ██║   ██║   ██║██╔══██╗██║╚██╔╝██║╚════██║
     ██║   ╚██████╔╝██║  ██║██║ ╚═╝ ██║███████║
     ╚═╝    ╚═════╝ ╚═╝  ╚═╝╚═╝     ╚═╝╚══════╝
    Decentralized P2P Compute Mesh for LLMs
```

**TURMS** is a decentralized, zero-trust peer-to-peer (P2P) inference dispatch engine designed to pool heterogeneous hardware—from macOS Apple Silicon devices to Linux NVIDIA/Intel rigs—into a dynamic, self-healing LLM compute mesh.

Instead of relying on heavy distributed state machines, central trackers, corporate clouds, or static orchestrators, TURMS uses lightweight POSIX primitives, hardware-aware dispatch algorithms, and direct Iroh QUIC transport (with Tailcat WireGuard fallbacks) to route queries directly to the optimal peer node.

---

## ⚡ Core Operators

Nodes and operators in the mesh are identified strictly by their network handles and cryptographic public keys.

| Handle / Alias | Node ID / Public Key | Hardware Grid | System Domain |
| :--- | :--- | :--- | :--- |
| **`@Osiris`** | `9b872725...` | `Alnitak` (Linux Ultra 7) | Core Architecture & Transport Routing |
| **`@Isis`** | `f58ce3c4...` | `Sirius` (Linux Intel) | Hardware Introspection & System Daemon |
| **`@Horus`** | `f58ce3c4...` | `Meskhetyu` (MacOS M5) | Edge Integration & Client SDKs |
---

## Key Features

* **Trackerless P2P Discovery:** Peer discovery occurs directly over Iroh QUIC via custom ALPN protocols (`/turms/discovery/1`) and Pkarr/DHT resolution. No central HTTP tracker or database required.

* **Hardware-Aware 3-Tier Scheduling:**
  * **Tier 1 (VRAM):** Instant execution (0s cold start) on peers with models pre-loaded in memory.
  * **Tier 2 (Disk/Cache):** Fast local reloads (~2–5s) from NVMe/SSD storage.
  * **Tier 3 (Network Download):** Guarded fallback fetch with automatic disk space validation.

* **Apple Silicon UMA Protection:** `psyche-hardware` enforces a 60% VRAM cap on Unified Memory Architectures (UMA), preventing system instability, OS memory pressure, and swap thrashing.

* **Cryptographic Node Identity:** Every node generates its own persistent Ed25519 key pair (`~/.config/turms/node_key.pem`). Routing operates via public Node IDs—no real names, IP addresses, Tailscale keys, or local file paths are exposed in public descriptors.

* **Dual-Transport Fallback:** Primary P2P transport uses Iroh QUIC; fallback transport wraps Tailcat WireGuard encrypted tunnels (`tc+` / `tcp`) for NAT traversal.

* **Real-Time Streaming & Dynamic Port Probing:** SSE chunk streaming (`--stream`) over QUIC for minimal time-to-first-token (TTFT), plus automatic backend port probing (8080, 8001, 8000, 7860, 9000, 5000) for local model servers (initial tests with vLLM, llama.cpp, MLX LM).

---

## Sample soup

```
                    ┌──────────────────────────────────────────┐
                    │         Turms Mesh (P2P)                 │
                    │                                          │
  ┌─────────┐       │  ┌──────────┐   ┌──────────┐             │
  │ Node:   │◄──────┼──│ Node:    │   │ Node:    │             │
  │Meskhetyu│       │  │ Alnitak  │   │ Sirius   │             │
  │ (Mac)   │       │  │ (Linux)  │   │ (Linux)  │             │
  └────┬────┘       │  └────┬─────┘   └────┬─────┘             │
       │            │       │              │                   │
  ┌────┴────┐       │  ┌────┴─────┐  ┌─────┴────┐              │
  │ MLX LM  │       │  │ vLLM     │  │ llama-   │              │
  │  :8080  │       │  │ :8001    │  │ server   │              │
  └─────────┘       │  └──────────┘  │  :8080   │              │
                    │                └──────────┘              │
                    │                                          │
                    │  Each node runs:                         │
                    │  ┌───────────────────────────────────┐   │
                    │  │ turmsd daemon                     │   │
                    │  │  (--listen mode)                  │   │
                    │  │  - Iroh P2P listener              │   │
                    │  │  - Discovery ALPN (/turms/disc/)  │   │
                    │  │  - Inference ALPN (/turms/inf.)   │   │
                    │  │  - Proxies to local model server  │   │
                    │  └───────────────────────────────────┘   │
                    └──────────────────────────────────────────┘

  ┌─────────────────┐   ┌─────────┐   ┌────────────────────┐
  │  turmsd CLIENT  │──▶│ Iroh P2P │──▶│ turmsd --listen    │
  │  (dispatch)     │   │ (QUIC)   │   │ (proxy to model    │
  │                 │   │          │   │  server via HTTP)  │
  └─────────────────┘   └─────────┘   └────────────────────┘
```

---

## Repository Structure

```
.
├── Cargo.toml               # Workspace manifest
├── psyche-hardware/         # Hardware detection library (Apache-2.0)
│   ├── src/lib.rs           # HardwareInfo, detect(), suggest_config()
│   └── tests/
├── turms-transport/         # Routing engine & transport adapters
│   ├── src/
│   │   ├── router.rs        # TurmsRouter, scoring matrix, 3-tier dispatch
│   │   ├── iroh_transport.rs# Iroh QUIC adapter & listener
│   │   ├── tailcat_transport.rs # WireGuard tunnel fallback adapter
│   │   ├── discovery.rs     # Descriptor store parser & DHT re-keying
│   │   └── bin/turmsd.rs    # CLI daemon & client binary
│   └── tests/
└── turms/                   # Operational scripts & service manifests
    ├── discover-env.sh      # Mesh registrar & hardware discovery CLI
    ├── turms-ctl.sh         # Local model server management CLI
    ├── *.service / *.timer  # Linux systemd units
    └── *.plist              # macOS launchd agents
```

---

## Quick Start

### 1. Prerequisites

Ensure Rust (1.75+) and native build dependencies are installed on your platform:

```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y build-essential pkg-config libssl-dev curl python3 git

# macOS
export SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX15.sdk
```

### 2. Build the Workspace

```bash
cargo build --release
cargo test --workspace
```

---

## Setup & Deployment Guide

### Provider Node (Serving Models)

1. **Start your local model server** (e.g., llama.cpp server, vLLM, or MLX LM e.g. on port 8080 or 8001).

2. **Start `turmsd` in listener mode:**

```bash
# Automatically probes and forwards queries to your local model server
./target/release/turmsd --listen
```

3. **Generate your local descriptor:**

```bash
bash turms/discover-env.sh self
```

### Requester Node (Dispatching Queries)

1. **Exchange descriptors** with peer nodes via standard P2P discovery routines.

2. **Dispatch an inference prompt across the mesh:**

```bash
# Route query deterministically to the best peer node
./target/release/turmsd \
  --model "meta-llama/Llama-2-7b-chat-hf" \
  --params-b 7 \
  --connect \
  --prompt "Explain quantum computing in simple terms." \
  --stream
```

---

## CLI Reference

### `turmsd` Daemon & Client

```bash
# Dispatch query to mesh over Iroh P2P
turmsd --model <MODEL_ID> --params-b <PARAMS_IN_BILLIONS> --connect --prompt "Hello"

# Enable SSE streaming response output
turmsd --model gemma-4-12b --connect --prompt "Write a story" --stream

# Run in listener/proxy mode on a provider node
turmsd --listen

# Force a specific model server backend port (bypasses auto-probing)
turmsd --listen --force-port 8001

# Direct local execution bypass (no P2P network dispatch)
turmsd --model llama-3.1-7b --params-b 7 --local-addr 127.0.0.1:8001 --prompt "Hello"

# List available models served across all discovered mesh peers
turmsd --list-models -v
```

### `discover-env.sh` Management

```bash
# Register local node descriptor
discover-env.sh self

# List known mesh peers
discover-env.sh peers

# Inspect detailed hardware report as JSON
discover-env.sh hw-json
```

---

## Security Model

* **Pseudonymous Identity:** All contributor signatures and node manifests strictly use developer handles/nicknames (`@Osiris, `@Isis`, `@Horus`) and Ed25519 Node IDs.
* **Admin Plane:** Machine-level SSH management 
* **Data Plane (Mesh):** All mesh routing operates via signed, public Ed25519 Node IDs (`~/.config/turms/node_key.pem`).
* **Zero Privacy Leaks:** Peer descriptors contain no IP addresses, hostnames, absolute file paths, or private configuration metadata.
* **Cellular Permissions:** Requests can only interface through dedicated ALPN endpoints (`/turms/inference/1` and `/turms/discovery/1`).

---

## License

`psyche-hardware` and `turms-transport` are distributed under the terms of the [Apache License (Version 2.0)](LICENSE).
