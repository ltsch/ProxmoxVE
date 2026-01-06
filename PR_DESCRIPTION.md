# Tdarr: NVIDIA GPU Support

## Summary

Adds **NVIDIA GPU** hardware acceleration support to Tdarr LXC containers.

> [!IMPORTANT]
> This PR prioritizes the **native `var_gpu` and `setup_hwaccel`** for Intel/AMD iGPU support. This PR adds NVIDIA support for `/dev/nvidia*` devices in addition to the standard `/dev/dri` interface.

> [!NOTE]
> This PR modifies **two files**: `ct/tdarr.sh` and `install/tdarr-install.sh`.

## NVIDIA Requirements

| GPU Type | Device Interface | Handled By |
|----------|-----------------|------------|
| Intel/AMD iGPU | `/dev/dri` (render nodes) | Native `var_gpu` + `setup_hwaccel` |
| NVIDIA | `/dev/nvidia*` (proprietary) | **This PR** (explicit passthrough + version-matched drivers) |

NVIDIA GPU passthrough to LXC containers requires:
1. **Explicit device passthrough** — `/dev/nvidia*` devices are not part of `/dev/dri`  
2. **Exact version matching** — Container userspace libraries must match host kernel driver

Without this, applications attempting to use the GPU (e.g., handbrake, ffmpeg) will fail with a `Driver/library version mismatch` error when attempting to initialize the NVML or CUDA interface.

### Core Libraries & Transcoding

The following libraries are matched to the host version to enable hardware acceleration:

| Library | Role | Enables |
|---------|------|---------|
| `libcuda1` | CUDA Base | Communication with GPU hardware |
| `libnvcuvid1` | NVDEC | **Hardware Decoding** in FFmpeg/Handbrake |
| `libnvidia-encode1` | NVENC | **Hardware Encoding** in FFmpeg/Handbrake |
| `libnvidia-ml1` | NVML | Device monitoring and temperature/load reporting |

### Service Management

The PR implements robust systemd service management to ensure reliability and ease of use:

- **`tdarr-server.service`**: Manages the main Tdarr Server. Includes `ExecStartPre` to automatically run the `Tdarr_Updater` before every start.
- **`tdarr-node.service`**: Manages the Tdarr Node. Configured with `Requires=tdarr-server.service` to ensure the correct startup sequence.
- **Auto-Enable**: Both services are enabled to start automatically on container boot, replacing the manual or cron-based startup requirement.
- **Configure a server or node**: `tdarr-node` or `tdarr-server` services can be enabled or disabled per container.
---

## Changes

### `ct/tdarr.sh` (Host-Side)

| Change | Purpose |
|--------|---------|
| NVIDIA detection | Identifies driver version to offer optional pinning |
| Driver pinning | Optional `apt-mark hold` on host to prevent breaking updates |
| Help/Abort flow | Prevents "half-broken" installs if drivers are missing on host |
| Uses community packages | Delegates passthrough logic to `build.func` natives |

### `install/tdarr-install.sh` (Container-Side)

| Change | Purpose |
|--------|---------|
| Autonomous detection | Reads `/proc/driver/nvidia/version` (shared with host) |
| Version matching | Installs `libcuda1` etc. pinned to host version |
| **Verification tools** | **Installs `nvidia-smi` for status monitoring** |
| **Systemd Services** | **Configures robust `tdarr-server` and `tdarr-node` services** |
| Intel VA-API | Optional `intel-media-va-driver-non-free` for multi-GPU support |

---

## Architecture

```
┌─ ct/tdarr.sh (Proxmox Host) ─────────────────────────────┐
│  1. Detect NVIDIA driver / Pin host drivers (Optional)   │
│  2. Delegate passthrough to build.func (var_gpu="yes")   │
└────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─ install/tdarr-install.sh (Inside Container) ────────────┐
│  1. Detect driver version directly via /proc             │
│  2. Pin NVIDIA CUDA repo + Install matched packages      │
│  3. Install nvidia-smi for user verification             │
└────────────────────────────────────────────────────────────┘
```

---

## Testing

- Verified on Tdarr built on Debian 13 (Trixie) with NVIDIA host driver 590.48
- Verified on Proxmox 8.4.16/368e3c45c15b895c (running kernel: 6.8.12-17-pve)
- Confirmed `apt` correctly prioritizes pinned NVIDIA repo
- Confirmed services start cleanly

---

## Files Changed

- `ct/tdarr.sh` — NVIDIA detection and device passthrough
- `install/tdarr-install.sh` — Userspace package installation
