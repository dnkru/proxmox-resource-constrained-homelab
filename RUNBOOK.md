# Runbook: Rebuilding the Homelab From Scratch

This runbook recreates the lab described in the README on a Dell OptiPlex 7010 (4 GB RAM, 500 GB boot HDD, 1 TB storage HDD). Follow the steps in order. Each step ends with a check, so you know it worked before moving on.

**Address plan** (fixed addresses live below the router's DHCP pool):

| Address | Role |
|---|---|
| 10.0.0.34 | Proxmox VE host |
| 10.0.0.35 | Pi-hole (CT 100) |
| 10.0.0.36 | Docker host (CT 101) |

Adjust the addresses and gateway (`10.0.0.1` here) to match your own network.

---

## 0. Before you start

- Back up anything you need from both drives. The install erases the drive you pick.
- Download the Proxmox VE ISO from proxmox.com/en/downloads.
- Write it to a USB drive (8 GB or larger) with balenaEtcher or Rufus (DD mode if asked).
- Use a wired Ethernet connection.
- Router: in the gateway's Local IP settings, set the DHCP pool to start at `10.0.0.100` so fixed addresses below it never collide with automatic ones.

## 1. BIOS settings

1. Power on and press **F2** at the Dell logo.
2. Enable **Intel Virtualization Technology** (and **VT for Direct I/O** if listed).
3. Set SATA operation to **AHCI**.
4. Turn **Secure Boot** off.
5. Save and exit.

**Check:** pressing **F12** at the Dell logo shows the one-time boot menu with your USB drive listed.

## 2. Install Proxmox VE

1. Boot from the USB drive (F12, choose the UEFI entry) and select **Install Proxmox VE (Graphical)**.
2. **Target disk:** choose the **500 GB** drive. Open the dropdown and confirm the size shown (about 465 GiB) before continuing. Use **ext4** under Options (not ZFS, which uses too much RAM for 4 GB).
3. **Location and time zone:** set the correct time zone (for example `America/New_York`).
4. **Password and email:** set a strong root password and a real email address. Store the password in a password manager.
5. **Network:** hostname `pve.home.lan`, IP `10.0.0.34/24`, gateway `10.0.0.1`, DNS of your choice.
6. On the summary screen, re-check the disk, time zone, filesystem, and IP, then click **Install**.
7. When it reboots, remove the USB drive.

**Check:** the console shows a login prompt and a URL like `https://10.0.0.34:8006/`.

## 3. First login and updates

1. From another computer on the same network, open `https://10.0.0.34:8006`. Accept the self-signed certificate warning.
2. Log in as `root` (realm: Linux PAM). Dismiss the "no valid subscription" popup.
3. Go to **pve, Updates, Repositories**:
   - Disable both **enterprise** repositories (the `pve-enterprise` one and the `ceph` one).
   - Click **Add** and choose **No-Subscription**.
   - Leave the two Debian repositories enabled.
4. Go to **Updates**, click **Refresh** (it should end in `TASK OK`), then **Upgrade**. Type `Y` when asked.
5. Reboot from the top-right button.

**Check:** after the reboot, Refresh ends in `TASK OK` and the Updates list is empty.

## 4. Add the 1 TB drive as storage

1. Go to **pve, Disks** and confirm the 1 TB drive appears with no partitions. Note its device name.
2. Go to **Disks, Directory, Create: Directory**:
   - Disk: the 1 TB drive
   - Filesystem: `ext4`
   - Name: `storage1tb`
3. Go to **Datacenter, Storage**, select `storage1tb`, click **Edit**, and set **Content** to: Disk image, Container, ISO image, Container template, VZDump backup file.

**Check:** `storage1tb` appears under `pve` in the left sidebar.

## 5. Pi-hole container (CT 100)

### 5.1 Download a template
Open `storage1tb`, then **CT Templates, Templates**. Search `debian`, select the newest **debian-xx-standard** template, and click **Download**.

### 5.2 Create the container
Click **Create CT**:

| Tab | Setting |
|---|---|
| General | ID `100`, hostname `pihole`, **Unprivileged** checked, set a password |
| Template | `storage1tb`, the Debian template |
| Disks | `storage1tb`, `4` GiB |
| CPU | `1` core |
| Memory | `512` MiB, swap `512` MiB |
| Network | Bridge `vmbr0`, IPv4 **Static** `10.0.0.35/24`, gateway `10.0.0.1` |
| DNS | Defaults |
| Confirm | Check **Start after created** |

Then open the container's **Options** and enable **Start at boot**.

**Check:** in the container console, log in as `root` and run:
```
ping -c 3 1.1.1.1
ping -c 3 google.com
```
Both should reply.

### 5.3 Install Pi-hole
```
apt update
apt full-upgrade -y
apt install -y curl
curl -sSL https://install.pi-hole.net | bash
```
In the installer: accept the static IP notice, choose an upstream DNS provider, keep the default blocklist, enable query logging, and enable the web interface. Then set the admin password:
```
pihole setpassword
```

### 5.4 Disable Pi-hole's NTP client
An unprivileged container cannot set the clock, and it does not need to (it uses the host's clock). Without this, Pi-hole logs "Insufficient permissions":
```
pihole-FTL --config ntp.ipv4.active false
pihole-FTL --config ntp.ipv6.active false
pihole-FTL --config ntp.sync.active false
```

**Check:**
```
dig @10.0.0.35 google.com +short        # returns addresses
dig @10.0.0.35 doubleclick.net +short   # returns 0.0.0.0
```
Open `http://10.0.0.35/admin` (http, not https) and log in.

### 5.5 Point a test device at Pi-hole
Set one device's DNS manually to `10.0.0.35` (static IP settings on a phone, or network settings on a computer). Turn off VPNs, Private DNS, and browser "Secure DNS" for the test. Browse for a minute.

**Check:** the device's address appears in Pi-hole's **Query Log**.

**Note:** many ISP gateways (including Xfinity's) do not let you change the DNS the router hands out. In that case, set DNS per device. Do not add a public DNS as a "backup", or devices will skip Pi-hole at random.

## 6. Docker container (CT 101)

### 6.1 Create the container
Click **Create CT**:

| Tab | Setting |
|---|---|
| General | ID `101`, hostname `docker`, **Unprivileged** checked, set a password |
| Template | `storage1tb`, the Debian template |
| Disks | `storage1tb`, `16` GiB |
| CPU | `1` core |
| Memory | `1024` MiB, swap `512` MiB |
| Network | Bridge `vmbr0`, IPv4 **Static** `10.0.0.36/24`, gateway `10.0.0.1` |
| DNS | DNS server `10.0.0.35`, search domain blank |
| Confirm | Check **Start after created** |

### 6.2 Enable the features Docker needs
1. Open **101, Options, Features** and check **Nesting** and **keyctl**.
2. Enable **Start at boot**.
3. **Shut down** the container, wait for it to stop, then **Start** it. The features only apply after a full restart.

**Check:** `ping -c 3 google.com` works in the container console, and `10.0.0.36` appears in Pi-hole's Query Log.

### 6.3 Install Docker
```
apt update
apt full-upgrade -y
apt install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
docker run hello-world
```

**Check:** `hello-world` prints "Hello from Docker!"

## 7. Uptime Kuma

```
mkdir -p /opt/uptime-kuma
cd /opt/uptime-kuma

cat > compose.yaml <<EOF
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
EOF

docker compose up -d
docker ps
```

1. Open `http://10.0.0.36:3001`.
2. Choose **SQLite** as the database (lowest RAM use, no extra server needed).
3. Create an admin account.
4. Add monitors:
   - Ping `10.0.0.34` (Proxmox host)
   - Ping `10.0.0.35` (Pi-hole DNS)
   - HTTP(s) `http://10.0.0.35/admin` (Pi-hole dashboard)

**Check:** `docker ps` shows `uptime-kuma` as healthy, and all three monitors are green.

All Uptime Kuma data lives in `/opt/uptime-kuma/data`.

## 8. Backups

Snapshots are not available because `storage1tb` is a plain directory (ext4). Use backups instead.

1. For each container: **Backup, Backup now**.
2. Storage: `storage1tb`. Mode: **Stop**. Compression: **ZSTD**.
3. Wait for `TASK OK`.

**Check:** both backups appear under **storage1tb, Backups**.

Limitation: these backups sit on the same drive as the containers. They protect against bad configs and updates, not drive failure. Copy them to another device periodically.

### Restore test
1. Select a backup under **storage1tb, Backups** and click **Restore**.
2. Restore to a new container ID (for example `199`) so the working container is untouched. If asked, do **not** start it on the same IP at the same time as the original (an address conflict).
3. Confirm the restored container starts, then delete the test container.

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `apt update` fails on the host | Enterprise repositories enabled | Disable them, add No-Subscription (step 3) |
| Docker fails to start in the container | Nesting or keyctl missing, or no restart after enabling | Enable both, then shut down and start the container |
| Test device not in Pi-hole's log | Device bypasses Pi-hole | Turn off VPN, Private DNS, and browser Secure DNS; re-check the static DNS entry |
| First `dig` times out, retry works | Cold cache, first lookup reached the upstream | Normal |
| "Insufficient permissions" NTP error in Pi-hole | Unprivileged container cannot set the clock | Disable Pi-hole's NTP client (step 5.4) |
| "Snapshots not supported" | Directory storage cannot snapshot container disks | Use backups (step 8) |
| Cannot reach `https://10.0.0.34:8006` | Different network, or typed `http://` | Use the same network and type `https://` |
| Pi-hole admin page will not load | Typed `https://` | Use `http://10.0.0.35/admin` |

## 10. Rebuild order summary

BIOS, install Proxmox on the 500 GB drive, fix repositories and update, add the 1 TB drive as `storage1tb`, create CT 100 and install Pi-hole, create CT 101 with nesting and keyctl and install Docker, deploy Uptime Kuma, take backups, test a restore.
