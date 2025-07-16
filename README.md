# Jellyfin Media Server: System Preparation Guide

This guide outlines the essential steps to prepare your Ubuntu system for a Dockerized Jellyfin media server. It covers hardware and storage configuration, data protection, network settings, and automatic system updates, laying a robust foundation for your media stack.

-----

## 1\. System Hardware & Storage Configuration

This server utilizes a multi-drive setup for optimal performance, storage, and data protection:

  - **NVMe SSD (OS):** The primary drive for the Ubuntu operating system (`/`).
  - **SATA SSD (Cache/Downloads/Transcodes):** Mounted at `/mnt/cache`, this high-speed drive is dedicated to:
      - Active downloads (`/mnt/cache/nzbget/downloads`).
      - All Docker application configurations (`/mnt/cache/jellyfin-stack`).
      - **Hardware transcoding temporary files** (`/mnt/cache/transcodes`), used by Jellyfin and Tdarr for fast video processing.
  - **HDDs (Media Storage):** Multiple Hard Disk Drives are pooled together using MergerFS into `/mnt/storage` for final media storage.
  - **HDD (Parity):** A dedicated HDD stores SnapRAID parity data for data recovery.

### File System Table (`fstab`) Configuration

The `/etc/fstab` file persistently mounts your drives. Key mount points include:

  - `/` (OS on NVMe)
  - `/mnt/cache` (SATA SSD)
  - Individual data drives (e.g., `/mnt/data1`, `/mnt/data2`)
  - `/mnt/parity` (SnapRAID parity drive)
  - `/mnt/storage` (The MergerFS pool of your data drives)

-----

## 2\. Storage Management & Data Protection

### MergerFS (Drive Pooling)

**Purpose:** MergerFS combines multiple HDDs into a single, unified filesystem at `/mnt/storage`. This simplifies library management for all Docker services, presenting them with one large folder for all media.

### SnapRAID (Data Protection)

**Purpose:** SnapRAID provides snapshot-based data redundancy. It protects the media on your data drives against drive failure using parity information stored on `/mnt/parity`. It is automated via a `cron` job that runs `snapraid sync` and `snapraid scrub` periodically.

-----

## 3\. Firewall (UFW) & External Access (Tailscale Funnel)

This server employs a "secure-by-default" network configuration.

### How it Works

1.  **Deny All Incoming:** UFW is configured to `deny` all incoming traffic from the public internet by default. This drastically reduces the server's attack surface.
2.  **Allow Tailscale Network:** UFW explicitly allows all traffic coming in on the `tailscale0` interface. This means **any device on your private Tailscale network can access all service web UIs** (Sonarr, Radarr, Homepage, etc.) using the server's Tailscale IP and the correct port (e.g., `http://100.x.x.x:7878`).
3.  **Secure SSH:** SSH access (port 22) is also restricted to the Tailscale network, preventing public SSH brute-force attacks.
4.  **Expose Jellyfin via Funnel:** **Only Jellyfin** (port `8096`) is intentionally and securely exposed to the public internet using `tailscale funnel`. This provides public access without opening ports on your router or exposing your home IP address.

This setup achieves a perfect balance: all management interfaces are kept private and secure, while your primary media service (Jellyfin) remains accessible from anywhere.

### Current UFW Rules (`sudo ufw status verbose`)

```bash
Status: active
Default: deny (incoming), allow (outgoing), deny (routed)

To                          Action      From
--                          ------      ----
Anywhere on tailscale0      ALLOW IN    Anywhere
```

*(Simplified for clarity - rule allows all traffic, including SSH, on the tailscale0 interface)*

### Current Tailscale Funnel Configuration

Jellyfin is exposed publicly via a secure URL.

```bash
# Example output
https://your-server-name.tailscale.ts.net -> http://localhost:8096
```

-----

## 4\. Automatic System Updates with Unattended Upgrades

To keep the underlying Ubuntu operating system secure, `unattended-upgrades` is configured to automatically install security patches. This service runs independently of Docker and ensures the host system remains up-to-date without manual intervention, complementing Watchtower's role in managing container updates.
