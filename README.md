# Buntu Jelly Media Server Setup

This README serves as a comprehensive guide and personal notes for my Dockerized Jellyfin media server. It covers hardware, storage management, core application configurations, and network settings, aimed at streamlining future maintenance and troubleshooting.

## 1. System Hardware & Storage Configuration

**Drives:** This computer is equipped with 5 drives, strategically allocated for optimal performance and storage:
1.  **NVMe SSD (OS):** The primary drive for the Ubuntu operating system (`/`).
2.  **SATA SSD (Cache/Downloads):** A high-speed SATA SSD (mounted at `/mnt/cache`) dedicated to active downloads, temporary files, and application configurations that benefit from SSD speeds. This is crucial for hardlinking efficiency.
3.  **HDDs (3x - Media Storage):** Three larger Hard Disk Drives (`/mnt/data1`, `/mnt/data2`, `/mnt/data3`) for the final, long-term storage of media files. These are pooled together.
4.  **HDD (Parity):** A dedicated HDD (`/mnt/parity`) for SnapRAID parity data, enabling data recovery in case of media drive failures.

**File System Table (`fstab`):**
* Review drive mounts, their file systems, and mounting options for persistence across reboots. This is where `mergerfs` is also configured.
    ```bash
    sudo nvim /etc/fstab
    ```
* **Expected Mount Points:**
    * `/` (OS on NVMe)
    * `/mnt/cache` (SATA SSD for downloads/cache)
    * `/mnt/data1`, `/mnt/data2`, `/mnt/data3` (Individual HDDs for media data)
    * `/mnt/parity` (HDD for SnapRAID parity)
    * `/mnt/storage` (MergerFS pooled media library)

## 2. Storage Management & Data Protection

### MergerFS (Drive Pooling)

* **Purpose:** Pools the multiple HDDs (`/mnt/data1`, `/mnt/data2`, `/mnt/data3`) into a single, unified virtual filesystem (`/mnt/storage`). This makes all your media appear as one large drive to Jellyfin and the "arr" applications, simplifying library management.
* **Configuration:** The MergerFS mount point (`/mnt/storage`) is defined and managed within `/etc/fstab`.
    ```bash
    sudo nano /etc/fstab
    ```
    (Look for the `mergerfs` entry that combines your individual data drives into `/mnt/storage`).

### SnapRAID (Data Protection)

* **Purpose:** Provides data redundancy and protection against drive failures (up to the number of parity drives configured). It's used to recover data from failed data drives by using parity information stored on a separate drive (`/mnt/parity`). It is typically run periodically.
* **Configuration:** The main SnapRAID configuration file:
    ```bash
    sudo nano /etc/snapraid.conf
    ```
    *Ensure `content` and `parity` paths are correct, and all `data` drives are listed.*
* **Automation:** SnapRAID synchronization and scrubbing are automated via a cron job to ensure data protection is regularly updated and verified.
    ```bash
    sudo crontab -e
    ```
    (Look for the `snapraid sync` and `snapraid scrub` commands).

## 3. Core Media Server Applications (Dockerized Stack)

All core media services are deployed as Docker containers, orchestrated using `docker-compose.yml`. This setup ensures portability, isolation, and simplified management.

**Docker Compose File Location:**
* The primary `docker-compose.yml` defining the entire stack is located in the `jellyfin-stack` directory within your home folder:
    ```bash
    cd ~/jellyfin-stack
    ```

**Consistent Permissions (`PUID` & `PGID`):**
* All Docker containers are configured with `PUID=1000` (for user `jevonx`) and `PGID=1001` (for group `mediamanagers`). This ensures uniform file ownership and permissions between the host system and the Docker containers, which is critical for read/write access and hardlinking.
* **Verify/Set Host Directory Permissions:**
    ```bash
    sudo chown -R jevonx:mediamanagers /mnt/cache/nzbget/downloads
    sudo chmod -R 775 /mnt/cache/nzbget/downloads
    sudo chown -R jevonx:mediamanagers /mnt/storage
    sudo chmod -R 775 /mnt/storage
    # Also for config directories if needed:
    sudo chown -R jevonx:mediamanagers ~/jellyfin-stack/jellyfin/config
    sudo chmod -R 775 ~/jellyfin-stack/jellyfin/config
    # ... and similarly for sonarr/radarr/lidarr/prowlarr/nzbget config directories.
    ```

**Crucial Docker Volume Mappings:**
These mappings connect host directories to container directories, making data accessible and persistent:
* **Download & Temporary Processing:** `/mnt/cache/nzbget/downloads` (Host - SATA SSD) maps to `/downloads` (Container) for NZBGet, Sonarr, Radarr, and Lidarr. **This is the source for hardlinks.**
* **Final Media Library:** `/mnt/storage` (Host - MergerFS pooled HDDs) maps to `/media` (Container) for Sonarr, Radarr, Lidarr, and Jellyfin. **This is the destination for hardlinks and the source for Jellyfin's libraries.**
* **Application Configurations:** `./{app}/config` (Host - within `~/jellyfin-stack`) maps to `/config` (Container) for each application's persistent settings.

**Docker Management Commands:**
* **Start the entire stack (detached mode):**
    ```bash
    cd ~/jellyfin-stack
    docker compose up -d
    ```
* **Stop and remove all services:**
    ```bash
    cd ~/jellyfin-stack
    docker compose down
    ```
* **Check status of running services:**
    ```bash
    docker compose ps
    ```
* **View real-time logs for a specific service (e.g., nzbget):**
    ```bash
    docker compose logs -f nzbget
    ```

### 3.1. Application-Specific Configurations

#### a. NZBGet (Usenet Download Client)

* **Docker Container Name:** `nzbget`
* **Web UI Access:** `http://your-server-ip:6789`
* **Host Path for Downloads:** `/mnt/cache/nzbget/downloads` (for incomplete, complete, and intermediate files)
* **Container Path for Downloads:** `/downloads`
* **Internal UI Paths (Settings -> Paths):**
    * `MainDir: /downloads`
    * `DestDir: /downloads/complete`
    * `InterDir: /downloads/incomplete`
* **Categories:** Ensure `tv`, `movies`, and `music` categories are created in the NZBGet UI. For each category, set the `DestDir` to just the category name (e.g., `tv` for the TV category), so NZBGet places files into `/downloads/complete/tv`, etc.
* **Usenet Server:** Configured with provider details (host, port, username, password, SSL).

#### b. Prowlarr (Indexer Management)

* **Docker Container Name:** `prowlarr`
* **Web UI Access:** `http://your-server-ip:9696`
* **Role:** Centralizes and manages Usenet indexers (and optionally Torrent trackers). It provides these indexers to Sonarr, Radarr, and Lidarr.
* **Configuration:**
    * Add Usenet indexers (e.g., NZBGeek) in Prowlarr.
    * Add Sonarr, Radarr, and Lidarr as "Apps" in Prowlarr, providing their correct URLs and API keys.
    * Perform a "Sync Indexers" action in Prowlarr after configuring "arr" apps to push indexers.

#### c. Sonarr / Radarr / Lidarr ("Arr" Apps - Automation)

* **Docker Container Names:** `sonarr`, `radarr`, `lidarr`
* **Web UI Access:**
    * Sonarr: `http://your-server-ip:8989`
    * Radarr: `http://your-server-ip:7878`
    * Lidarr: `http://your-server-ip:8686`
* **Key Settings for Each App:**
    * **Download Client (NZBGet):** Configured as the primary download client for each "arr" app.
    * **Remote Path Mappings (CRITICAL for Hardlinking):**
        * This mapping tells the "arr" app how to find a completed download that NZBGet reports.
        * Example for Sonarr:
            * `Remote Path:` `/downloads/complete/tv` (This is what NZBGet tells Sonarr)
            * `Local Path:` `/downloads/complete/tv` (This is where Sonarr finds the file within its own container, using the *same* mapped volume as NZBGet).
        * Repeat this pattern for Radarr (`/downloads/complete/movies`) and Lidarr (`/downloads/complete/music`).
    * **Media Management:** "Use Hardlinks Instead Of Copy" **must be enabled** in each app's Media Management settings. This ensures efficient, space-saving imports.
    * **Root Folders:** These define where the final media files are stored on your pooled storage:
        * Sonarr: `/media/tv`
        * Radarr: `/media/movies`
        * Lidarr: `/media/music`
        (These align with the `/mnt/storage:/media` Docker volume mapping).

#### d. Jellyfin (Media Server)

* **Docker Container Name:** `jellyfin`
* **Web UI Access:** `http://your-server-ip:8096`
* **Media Library Paths:** Configured in the Jellyfin UI to point to the correct container paths for your media:
    * Add a Movies library, path: `/media/movies`
    * Add a TV Shows library, path: `/media/tv`
    * Add a Music library, path: `/media/music`
* **Hardware Transcoding:** Enabled using `/dev/dri` device mapping in the `docker-compose.yml` for Intel iGPU acceleration. Verify in Jellyfin's Dashboard -> Transcoding settings.

## 4. Usenet Providers & Indexers

### Usenet Provider
* **Current Provider:** Usenet Farm 

### Usenet Indexer
* **Current Indexer:** NZBGeek 
## Firewall (UFW) & External Access (Tailscale Funnel) Configuration

This section details the Uncomplicated Firewall (UFW) settings and how Tailscale Funnel is used to manage external access to services on this server.

### 1. Purpose of this Configuration

The primary goals of this firewall and access configuration are:

* **Enhanced Security:** To significantly reduce the server's attack surface by blocking all incoming connections by default.
* **Restricted Internal Access:** To ensure that all services (like Sonarr, Radarr, NZBGet, Lidarr, Readarr, etc.) are only accessible from devices within the private Tailscale network. This means they are *not* directly exposed to the public internet.
* **Secure External Access for Jellyfin:** To provide secure, encrypted, and easily managed public access to the Jellyfin server specifically, without exposing the server's public IP or requiring complex router port forwarding.

### 2. How this Configuration was Achieved

This setup was configured by:

* **Resetting UFW:** The existing UFW rules were reset to their default, highly restrictive state (deny incoming, allow outgoing).
* **Allowing Tailscale Network Traffic:** A specific UFW rule was added to permit all incoming traffic originating from the `tailscale0` network interface. This allows any device authenticated to this server's Tailscale network to access all services running on the server.
* **Restricting SSH:** The SSH service (port 22) was explicitly configured to only accept connections originating from within the Tailscale network. Public SSH access is blocked.
* **Enabling Tailscale Funnel for Jellyfin:** The `tailscale funnel` command with the `--bg` (background/persistent) flag was used to expose the Jellyfin web interface (on port 8096) to the public internet via a secure Tailscale relay.

### 3. Current UFW Rules (`sudo ufw status verbose`)

Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From

22/tcp on tailscale0       ALLOW IN    Anywhere

Anywhere on tailscale0     ALLOW IN    Anywhere

22/tcp (v6) on tailscale0  ALLOW IN    Anywhere (v6)

Anywhere (v6) on tailscale0 ALLOW IN    Anywhere (v6)

* **`Default: deny (incoming)`:** This is the core security principle. Nothing comes in unless explicitly allowed.
* **`Anywhere on tailscale0 ALLOW IN Anywhere`:** This rule (and its IPv6 counterpart) grants full access to the server from any device connected to the Tailscale network. This ensures all your *arrs, NZBGet, and other internal services are accessible from your trusted devices.
* **`22/tcp on tailscale0 ALLOW IN Anywhere`:** This rule specifically permits SSH access *only* from devices on your Tailscale network.

### 4. Current Tailscale Funnel Configuration (`sudo tailscale funnel status`)

This command shows which services are currently being exposed to the public internet via Tailscale Funnel.

```
# Example output (your server name will differ)
[https://your-server-name.tailscale.ts.net:8096/](https://your-server-name.tailscale.ts.net:8096/)  / proxy [http://127.0.0.1:8096](http://127.0.0.1:8096)
```
---

### 5. Stop specific Funnel (Jellyfin port 8096)

`sudo tailscale funnel --bg 8096 off`

#### Turn off all funnels 

`sudo tailscale funnel --off-all`

#### Funnel status

`sudo tailscale funnel status`

```Bash
sudo systemctl stop tailscaled
```
Start the Tailscale Daemon:

```Bash
sudo systemctl start tailscaled
```

Restart the Tailscale Daemon:

```Bash
sudo systemctl restart tailscaled
```

Disconnect from Tailscale Network:

```Bash
sudo tailscale down
```

**Last Updated:** June 24
