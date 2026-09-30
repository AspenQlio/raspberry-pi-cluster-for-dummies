# Raspberry Pi Cluster for Dummies

> Objective: Decouple a saturated HomeLab running on a single Raspberry Pi (1GB RAM) into a two-node architecture, separating mission-critical services (DNS/Security) from heavy workloads (Media/Storage).

## 1. The Original Problem

The primary server (Raspberry Pi 3, 1GB RAM) was on the verge of collapse. The symptoms were clear:
- Constant load average above `14.00`
- I/O Wait usage hovering around `66%`
- Total saturation of ZRAM space (955MB at 100%) and massive usage of the Swapfile on the MicroSD card (very slow).

The bottleneck was caused by trying to run **Frigate NVR** (CPU-based AI processing) and **Jellyfin** alongside the core infrastructure (**Pi-hole**, **Home Assistant**, **Vaultwarden**). The lack of physical RAM forced the kernel to use the MicroSD as RAM (Swap), causing system processes to compete for I/O time.

## 2. Decoupled Architecture

To avoid conflicts and ensure network stability (local DNS), the workload was split into two nodes, communicating over the **Tailscale** VPN network (`100.x.x.x`).

```text
[Internet]
    |
[Router] (Assigns IPs 192.168.100.x)
    |
    |---> Node 1 (The Brain) --- IP: 192.168.100.75 | Tailscale: 100.84.189.38
    |       • Hardware: Raspberry Pi (1GB RAM) + 60GB MicroSD
    |       • Services: Pi-hole, Unbound, Home Assistant, Homepage, Uptime Kuma, Syncthing, Tailscale (Exit Node).
    |       • Objective: High availability (Mission critical). Must never turn off.
    |
    |---> Node 2 (The Muscle) --- IP: 192.168.100.80 | Tailscale: 100.66.109.86
            • Hardware: Raspberry Pi (1GB RAM) + External USB Hub + 1TB SSD
            • Services: Jellyfin, Transmission, Subliminal, Prowlarr, Samba (SMB), Dockge, Watchtower.
            • Objective: Media processing and 24/7 downloads.
```

## 3. Service Deployment (How it was done)

### On Node 1 (The Brain)

**1. Unbound (Extreme Privacy DNS):**
Instead of relying on Google or Cloudflare, Unbound was installed to act as our own recursive DNS resolver.
```bash
sudo apt-get install unbound
```
`/etc/unbound/unbound.conf.d/pi-hole.conf` was configured on port `5335`, and Pi-hole v6 configuration (`/etc/pihole/pihole.toml`) was set to exclusively use `127.0.0.1#5335` as its upstream.

**2. Tailscale Exit Node:**
To allow mobile devices to block ads via Pi-hole while on public networks (4G/5G), IP forwarding was enabled in the kernel:
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
sudo tailscale up --advertise-exit-node --ssh
```

### On Node 2 (The Muscle)

**1. Permanent mounting of the SSD (1TB):**
The UUID of the `/dev/sda2` partition was identified to avoid drive letter dependencies, and its automatic mount point at `/srv/media` was configured via `/etc/fstab`:
```text
UUID=726b4d76-cdd5-4a51-ac5e-6a40baabf715 /srv/media ext4 defaults,noatime 0 2
```

**2. Samba (Network Drive):**
To natively manage and send files to the 1TB SSD from any laptop in the house (Windows/Linux/Mac), `samba` was installed and the media directory was exported:
```bash
sudo apt-get install samba
sudo smbpasswd -a aspencito
```

**3. Jellyfin, Transmission, and Prowlarr:**
Deployed via `docker-compose.yaml` (managed through **Dockge** on port `5001`). Transmission was configured with access to `/srv/media/downloads` and Jellyfin to `/srv/media/movies` and `/srv/media/series`.
Prowlarr is linked to Transmission to inject automated torrent searches. All of this forms the "Zero-Clicks" strategy.

**4. Watchtower (Automatic Updates):**
The `containrrr/watchtower` container was added with a cron schedule (`0 0 4 * * *`) that checks for updates at 4:00 AM, downloads new Jellyfin or Transmission images, and restarts them automatically to keep the server secure.

## 4. Errors during migration and how they were solved

### Error 1: SSD dependency for booting Node 1
When trying to remove the 1TB SSD from Node 1 to hand it over to Node 2, we realized the entire OS was running from the `/dev/sda2` partition on the SSD, and the MicroSD was only acting as a bootloader.
**Solution:** The root partition of the MicroSD (`/dev/mmcblk0p2`) was mounted, `rsync` was used to copy recent data (Docker, Vaultwarden, Pi-hole, Tailscale state), and `/boot/firmware/cmdline.txt` was edited to restore `root=PARTUUID=1c0f53ad-02`, pointing back to the MicroSD. This allowed safe disconnection of the SSD.

### Error 2: Jellyfin lacking permissions to read movies
On Node 2, the movies were located on the SSD under a private directory with restrictive permissions (`700`).
**Solution:** Public paths `/srv/media/movies` and `/srv/media/series` were created with `755` permissions, and the catalog was moved to this new hierarchy, allowing the Jellyfin container to read the mapped volume without ACL issues.

### Error 3: "Undervoltage" danger on Node 2
Connecting the 1TB SSD directly to the USB port of the Raspberry Pi 2 exceeds the board's current capacity (~1.2A), which would cause random reboots or data corruption.
**Solution:** A USB Hub with an independent external power supply was included. The SSD draws power directly from the wall outlet, allowing the Raspberry Pi to use 100% of its own power supply for processing.

## 5. Automatic subtitles without APIs
Using native Jellyfin or OpenSubtitles extensions was discarded because they require user registration and enforce download quotas.
Instead, native `subliminal` was installed on the host environment of the Raspberry Pi 2. A `cron` job runs the following script every hour on the hour:
```bash
#!/bin/bash
subliminal download -l es /srv/media/movies
subliminal download -l es /srv/media/series
```
It scans the directory for new video files and downloads the adjacent `.srt` without user intervention.

## 6. Automated Backups
The configuration files for Home Assistant and Pi-hole are critical. A `cron` script (`backup.sh`) runs at 3:00 AM on Node 1:
1. Generates a `.tar.gz` of the Docker configuration and `/etc/pihole`.
2. Uses `scp` to send an encrypted copy via Tailscale (`100.66.109.86`) to the `/srv/media/backups` folder on the 1TB SSD on Node 2.
3. Uses `rclone` to upload a copy to Google Drive.
This ensures both physical redundancy (secondary SSD) and off-site redundancy (Cloud).
