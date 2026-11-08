# Platform Compatibility Guide: macOS (Apple Silicon) & Ubuntu (Intel x86_64)

This lab suite is designed to be fully portable across development environments. Whether running on **macOS (Apple Silicon / ARM64)** or a bare-metal **Ubuntu (Intel / AMD64)** workstation, the exact same `docker-compose.yml` definitions, networking commands, and container configurations will execute without modification.

This guide outlines how each platform handles the underlying virtualization and highlights small platform-specific prerequisites to ensure smooth execution.

---

## Architecture Compatibility Matrix

All primary container images used across the curriculum are official multi-architecture images. Docker automatically pulls the binary matched to your host CPU architecture:

| Component / Image | macOS (Apple Silicon ARM64) | Ubuntu Linux (Intel x86_64) | Architecture Support |
| :--- | :--- | :--- | :--- |
| **Diagnostics (`nicolaka/netshoot`)** | Native ARM64 | Native AMD64 | Multi-arch manifest |
| **L4 Balancer (`haproxy:alpine`)** | Native ARM64 | Native AMD64 | Multi-arch manifest |
| **L7 Reverse Proxy (`nginx:alpine`)** | Native ARM64 | Native AMD64 | Multi-arch manifest |
| **DNS Resolver (`coredns/coredns`)** | Native ARM64 | Native AMD64 | Multi-arch manifest |
| **Object Storage (`azure-storage/azurite`)** | Native ARM64 | Native AMD64 | Multi-arch manifest |
| **Database (`azure-sql-edge`)** | Native ARM64 | Native AMD64 | Dual-arch (ARM64 + AMD64) |
| **SMB File Share (`dperson/samba`)** | Native ARM64 | Native AMD64 | Multi-arch manifest |

---

## Platform Architectural Differences

### 1. macOS (Docker Desktop / Colima VM Architecture)
* **How it works:** macOS does not natively support Linux cgroups or network namespaces. Docker runs inside a lightweight, managed Linux utility VM.
* **Network Isolation:** Any routing modifications (`ip route add`), kernel parameter updates (`sysctls: net.ipv4.ip_forward=1`), or firewall rules (`iptables`) execute strictly within the virtualized Linux subsystem. Your physical macOS network interfaces (`en0`, Wi-Fi) and host routing tables are completely insulated and untouched.
* **Database Selection:** We use `mcr.microsoft.com/azure-sql-edge` because it was built specifically by Microsoft for native execution on ARM64 processors (Apple Silicon), avoiding the severe performance penalties of x86 Rosetta emulation.

### 2. Ubuntu Linux (Native Bare-Metal Kernel Architecture)
* **How it works:** Docker runs directly on the bare-metal Linux kernel. Containers share the host kernel via native namespaces, cgroups, and Linux software bridges (`br-xxxx`).
* **Performance:** Network packet processing, `conntrack` state lookups, and `iptables` traversals run with native hardware performance and near-zero latency overhead.
* **Host vs. Container Scope:** Commands executed inside containers using `cap_add: [NET_ADMIN]` configure the container's isolated network namespace, preserving host stability while executing with native kernel speed.

---

## Ubuntu-Specific Prerequisites

While macOS handles virtualization drivers automatically within the Docker VM, bare-metal Ubuntu hosts require two quick checks:

### 1. Enable CIFS Kernel Module (Required for Lab 5.3: Azure Files)
Lab 5.3 mounts an SMB/CIFS filesystem inside Linux containers (`mount -t cifs`). On Ubuntu, the container relies on the host kernel's CIFS filesystem driver. 

Run this once on your Ubuntu host terminal before launching Lab 5.3:

```bash
# Verify if CIFS is loaded
lsmod | grep cifs

# If no output is returned, load the CIFS module into the kernel:
sudo modprobe cifs
```

To persist this across host reboots:
```bash
echo "cifs" | sudo tee -a /etc/modules-load.d/cifs.conf
```

### 2. Docker Non-Root User Permissions
Ensure your Ubuntu user can run Docker commands without prepending `sudo`:

```bash
# Add your user to the docker group
sudo usermod -aG docker $USER

# Apply group changes immediately (or log out and back in)
newgrp docker
```

---

## Verifying Your Environment

Before starting any lab, confirm that Docker Engine and Compose V2 are available on your machine:

```bash
docker --version && docker compose version
```

### Expected Output
* **macOS:** `Docker version 24.x+`, `Docker Compose version v2.x+` (Apple Silicon architecture)
* **Ubuntu:** `Docker version 24.x+`, `Docker Compose version v2.x+` (x86_64 architecture)

Once verified, you can proceed directly to any lab directory and launch with standard Compose commands:
```bash
docker compose up -d
```