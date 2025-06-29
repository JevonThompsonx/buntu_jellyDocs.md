# Jellyfin Media Server: System Preparation Guide

This guide outlines the essential steps to prepare your Ubuntu system for a Dockerized Jellyfin media server. It covers hardware and storage configuration, data protection, network settings, and automatic system updates, laying a robust foundation for your media stack.

-----

## 1\. System Hardware & Storage Configuration

This server utilizes a multi-drive setup for optimal performance, storage, and data protection:

  * **NVMe SSD (OS):** The primary drive for the Ubuntu operating system (`/`).
  * **SATA SSD (Cache/Downloads):** Mounted at `/mnt/cache`, this high-speed drive is dedicated to active downloads, temporary files, and application configurations that benefit from SSD speeds. It's crucial for efficient hardlinking.
  * **HDDs (3x - Media Storage):** Three larger Hard Disk Drives (`/mnt/data1`, `/mnt/data2`, `/mnt/data3`) are used for the final, long-term storage of media files. These drives are pooled together using MergerFS.
  * **HDD (Parity):** A dedicated HDD (`/mnt/parity`) stores SnapRAID parity data, enabling data recovery in case one or more media drives fail.

### File System Table (`fstab`) Configuration

The `/etc/fstab` file manages how your drives are mounted and configured persistently across reboots. This is also where **MergerFS** is set up.

To review and edit your `fstab` file:

```bash
sudo nvim /etc/fstab
```

**Ensure the following mount points are configured:**

  * `/` (OS on NVMe)
  * `/mnt/cache` (SATA SSD for downloads/cache)
  * `/mnt/data1`, `/mnt/data2`, `/mnt/data3` (Individual HDDs for media data)
  * `/mnt/parity` (HDD for SnapRAID parity)
  * `/mnt/storage` (MergerFS pooled media library)

-----

## 2\. Storage Management & Data Protection

### MergerFS (Drive Pooling)

**Purpose:** MergerFS combines your multiple HDDs (`/mnt/data1`, `/mnt/data2`, `/mnt/data3`) into a single, unified virtual filesystem (`/mnt/storage`). This simplifies media library management by presenting all your drives as one large pool to Jellyfin and the "Arr" applications.

**Configuration:** The MergerFS mount point (`/mnt/storage`) is defined within `/etc/fstab`. Look for the `mergerfs` entry that combines your individual data drives into `/mnt/storage`.

```bash
sudo nano /etc/fstab
```

### SnapRAID (Data Protection)

**Purpose:** SnapRAID provides data redundancy and protection against drive failures (up to the number of parity drives configured). It allows you to recover data from failed data drives using parity information stored on a separate drive (`/mnt/parity`). SnapRAID is typically run periodically.

**Configuration:** The main SnapRAID configuration file is located at `/etc/snapraid.conf`.

```bash
sudo nano /etc/snapraid.conf
```

  * **Verify** that `content` and `parity` paths are correct, and all `data` drives are listed.

**Automation:** SnapRAID synchronization and scrubbing are automated via a cron job to ensure data protection is regularly updated and verified.

```bash
sudo crontab -e
```

  * Look for the `snapraid sync` and `snapraid scrub` commands in your crontab.

-----

## 3\. Firewall (UFW) & External Access (Tailscale Funnel) Configuration

This section details the Uncomplicated Firewall (UFW) settings and how Tailscale Funnel is used to manage external access to services on this server.

### 3.1. Purpose of this Configuration

The primary goals of this firewall and access configuration are:

  * **Enhanced Security:** To significantly reduce the server's attack surface by blocking all incoming connections by default.
  * **Restricted Internal Access:** To ensure that all services (like Sonarr, Radarr, NZBGet, Lidarr, Readarr, etc.) are only accessible from devices within the private Tailscale network. This means they are *not* directly exposed to the public internet.
  * **Secure External Access for Jellyfin:** To provide secure, encrypted, and easily managed public access to the Jellyfin server specifically, without exposing the server's public IP or requiring complex router port forwarding.

### 3.2. How this Configuration was Achieved

This setup was configured by:

  * **Resetting UFW:** The existing UFW rules were reset to their default, highly restrictive state (deny incoming, allow outgoing).
  * **Allowing Tailscale Network Traffic:** A specific UFW rule was added to permit all incoming traffic originating from the `tailscale0` network interface. This allows any device authenticated to this server's Tailscale network to access all services running on the server.
  * **Restricting SSH:** The SSH service (port 22) was explicitly configured to only accept connections originating from within the Tailscale network. Public SSH access is blocked.
  * **Enabling Tailscale Funnel for Jellyfin:** The `tailscale funnel` command with the `--bg` (background/persistent) flag was used to expose the Jellyfin web interface (on port 8096) to the public internet via a secure Tailscale relay.

```bash
# 1. Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed

# 2. Add your specific rules for Tailscale (IPv4 and IPv6)
sudo ufw allow in on tailscale0 to any port 22 proto tcp
sudo ufw allow in on tailscale0
sudo ufw allow in on tailscale0 to any port 22 proto tcp from ::/0
sudo ufw allow in on tailscale0 from ::/0

# 3. Enable UFW (if not already active)
# IMPORTANT: ONLY RUN THIS AFTER YOU'VE ADDED YOUR SSH ALLOW RULES!
sudo ufw enable

# 4. Configure logging
sudo ufw logging on

# 5. Verify your settings
sudo ufw status verbose
```

### 3.3. Current UFW Rules (`sudo ufw status verbose`)

```bash
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                                Action        From
--                                ------        ----
22/tcp on tailscale0              ALLOW IN      Anywhere
Anywhere on tailscale0            ALLOW IN      Anywhere
22/tcp (v6) on tailscale0         ALLOW IN      Anywhere (v6)
Anywhere (v6) on tailscale0       ALLOW IN      Anywhere (v6)
```

  * `Default: deny (incoming)`: This is the core security principle. Nothing comes in unless explicitly allowed.
  * `Anywhere on tailscale0 ALLOW IN Anywhere`: This rule (and its IPv6 counterpart) grants full access to the server from any device connected to the Tailscale network. This ensures all your "Arr" applications, NZBGet, and other internal services are accessible from your trusted devices.
  * `22/tcp on tailscale0 ALLOW IN Anywhere`: This rule specifically permits SSH access *only* from devices on your Tailscale network.

### 3.4. Current Tailscale Funnel Configuration (`sudo tailscale funnel status`)

This command shows which services are currently being exposed to the public internet via Tailscale Funnel.

```bash
# Example output (your server name will differ)
https://your-server-name.tailscale.ts.net:8096/  / proxy http://localhost:8096
```

### 3.5. Tailscale Funnel Management Commands

  * **Stop specific Funnel (Jellyfin port 8096):**
    ```bash
    sudo tailscale funnel --bg 8096 off
    ```
  * **Turn off all funnels:**
    ```bash
    sudo tailscale funnel --off-all
    ```
  * **Funnel status:**
    ```bash
    sudo tailscale funnel status
    ```
  * **Stop the Tailscale Daemon:**
    ```bash
    sudo systemctl stop tailscaled
    ```
  * **Start the Tailscale Daemon:**
    ```bash
    sudo systemctl start tailscaled
    ```
  * **Restart the Tailscale Daemon:**
    ```bash
    sudo systemctl restart tailscaled
    ```
  * **Disconnect from Tailscale Network:**
    ```bash
    sudo tailscale down
    ```

-----

## 4\. Automatic System Updates with Unattended Upgrades

To ensure your server remains secure and up-to-date with the latest security patches and bug fixes, it's highly recommended to enable automatic system updates using `unattended-upgrades`. This service is designed for safe and non-interactive upgrades of critical packages.

### Why Use `unattended-upgrades`?

  * **Security:** Automatically applies security updates, patching vulnerabilities quickly.
  * **Stability:** Designed to handle updates gracefully, minimizing the risk of system disruption.
  * **Dependency Management:** Intelligently resolves package dependencies during upgrades.
  * **Reboot Management:** Can be configured to automatically reboot the server at a specified time if required (e.g., after a kernel update).
  * **Logging:** Provides detailed logs of update activity.

### Setup Instructions

Follow these steps to configure `unattended-upgrades` on your Ubuntu or Debian server:

1.  **Update Package Lists and Install `unattended-upgrades`:**
    First, ensure your package lists are up-to-date and install the `unattended-upgrades` package if it's not already present.

    ```bash
    sudo apt update
    sudo apt install unattended-upgrades -y
    ```

2.  **Enable Automatic Upgrades Interactively:**
    Run the `dpkg-reconfigure` command to go through an interactive setup. When prompted, select `Yes` to enable automatic updates. This will create or update the `/etc/apt/apt.conf.d/20auto-upgrades` configuration file.

    ```bash
    sudo dpkg-reconfigure --priority=low unattended-upgrades
    ```

3.  **Customize Unattended Upgrades (Optional but Recommended):**
    The main configuration for `unattended-upgrades` is located in `/etc/apt/apt.conf.d/50unattended-upgrades`. Open this file with your preferred text editor (e.g., `nano`):

    ```bash
    sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
    ```

    Inside this file, you can make the following common customizations by uncommenting (`//` to nothing) or modifying the relevant lines:

      * **Enable Regular Updates (not just security):**
        By default, it often only applies security updates. To include general package updates (bug fixes, minor features), uncomment the `"${distro_id}:${distro_codename}-updates";` line:

        ```diff
        --- a/etc/apt/apt.conf.d/50unattended-upgrades
        +++ b/etc/apt/apt.conf.d/50unattended-upgrades
        @@ -8,7 +8,7 @@
          Unattended-Upgrade::Allowed-Origins {
                   "${distro_id}:${distro_codename}";
                   "${distro_id}:${distro_codename}-security";
        -//       "${distro_id}:${distro_codename}-updates";
        +//       "${distro_id}:${distro_codename}-updates";
                   "${distro_id}:${distro_codename}-proposed";
                   "${distro_id}:${distro_codename}-backports";
           };
        ```

        (Remove the `//` from `"${distro_id}:${distro_codename}-updates";`)

      * **Automatic Reboot (with Time):**
        If you want the server to automatically reboot when a reboot is required (e.g., after a kernel update), uncomment and set `Automatic-Reboot` to `"true"`. You can also specify a time for the reboot (e.g., "03:00" for 3 AM) to minimize disruption.

        **Caution:** Be mindful when enabling automatic reboots for production services that might require specific startup sequences or manual checks.

        ```diff
        --- a/etc/apt/apt.conf.d/50unattended-upgrades
        +++ b/etc/apt/apt.conf.d/50unattended-upgrades
        @@ -56,8 +56,8 @@
          // Automatically reboot WITHOUT CONFIRMATION if
          // the file /var/run/reboot-required is found after the upgrade
          //Unattended-Upgrade::Automatic-Reboot "false";
        -// If automatic reboot is enabled and needed, reboot at the specific
        -// time instead of immediately
        -// Default: "now"
        -//Unattended-Upgrade::Automatic-Reboot-Time "02:00";
        +// Unattended-Upgrade::Automatic-Reboot "true";
        +// Unattended-Upgrade::Automatic-Reboot-Time "03:00"; // Example: reboot at 3 AM
        ```

        (Uncomment `Unattended-Upgrade::Automatic-Reboot "true";` and `Unattended-Upgrade::Automatic-Reboot-Time "03:00";`)

      * **Email Notifications:**
        To receive email notifications in case of problems during an upgrade or if a reboot is needed, uncomment `Unattended-Upgrade::Mail` and replace `"root@localhost"` with your email address. Note that this requires a mail transfer agent (MTA) like Postfix or Sendmail to be installed and configured on your server.

        ```diff
        --- a/etc/apt/apt.conf.d/50unattended-upgrades
        +++ b/etc/apt/apt.conf.d/50unattended-upgrades
        @@ -20,7 +20,7 @@
          // Send email to this address for problems or new releases
          //Unattended-Upgrade::Mail "root@localhost";
        ```

        (Uncomment and change to `Unattended-Upgrade::Mail "your_email@example.com";`)

      * **Remove Unused Dependencies:**
        To automatically clean up orphaned packages after an upgrade (similar to `apt autoremove`), uncomment `Unattended-Upgrade::Remove-Unused-Dependencies`:

        ```diff
        --- a/etc/apt/apt.conf.d/50unattended-upgrades
        +++ b/etc/apt/apt.conf.d/50unattended-upgrades
        @@ -67,7 +67,7 @@
          // Do automatic removal of new unused dependencies after the upgrade
          // (equivalent to apt-get autoremove)
          //Unattended-Upgrade::Remove-Unused-Dependencies "false";
        +//Unattended-Upgrade::Remove-Unused-Dependencies "true";
        ```

        (Uncomment `Unattended-Upgrade::Remove-Unused-Dependencies "true";`)

      * **Save and Exit:** Save the file (Ctrl+O, then Enter with `nano`) and exit (Ctrl+X with `nano`).

4.  **Verify Configuration (Optional):**
    You can check the daily frequency of updates by inspecting the `20auto-upgrades` file:

    ```bash
    cat /etc/apt/apt.conf.d/20auto-upgrades
    ```

    You should see lines like `APT::Periodic::Update-Package-Lists "1";` and `APT::Periodic::Unattended-Upgrade "1";`, indicating daily execution.

5.  **Perform a Dry Run (Optional):**
    To simulate what `unattended-upgrades` would do without actually applying changes, run a dry run:

    ```bash
    sudo unattended-upgrades --dry-run --debug
    ```

    This command will output detailed information about which packages would be upgraded.

### Monitoring

You can monitor the activity of `unattended-upgrades` by checking its logs:

```bash
sudo tail -f /var/log/unattended-upgrades/unattended-upgrades.log
```

Or view the entire log file:

```bash
sudo less /var/log/unattended-upgrades/unattended-upgrades.log
```

-----
