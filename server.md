# AMD Family Server — Beginner-Friendly Step-by-Step Build Guide

## What this guide fixes

The earlier guide described **what** to build, but it sometimes skipped the small operational steps:

- How to find your server's IP address and MAC address.
- How to tell whether Ubuntu is Desktop or Server.
- How to identify the 1TB USB disk without accidentally formatting the wrong disk.
- Exactly which file to edit, how to open it, what to paste, and how to save it.
- How to test each step before moving on.
- What output you should expect.
- What to do when a command fails.
- How to connect from Windows, Android, an iPhone, a TV, or another laptop.
- How to verify that a container actually starts and stays healthy.
- How to undo changes safely.

This version is intentionally slower and more explicit. **Do not skip the checkpoints.** On a small 4GB machine, adding services one by one and testing them is much easier than installing everything at once and debugging five problems together.

---

# 1. Your target setup

## Hardware

Based on the original plan, this server has:

- AMD A6 PRO-7400B
- 2 CPU cores / 2 threads
- 4GB RAM
- About 218GB internal disk
- 1TB external USB HDD
- Ubuntu already installed
- Wired network or reliable Wi-Fi
- 24/7 power

## Services we will build

The final layout will look like this:

```text
                         Internet
                            |
                     Tailscale VPN
                            |
                   +------------------+
                   | AMD A6 Server    |
                   | Ubuntu           |
                   | Docker           |
                   +------------------+
                     |    |    |    |
                     |    |    |    +-- Uptime Kuma
                     |    |    +------- Pi-hole
                     |    +------------ Jellyfin
                     +----------------- Syncthing
                     +----------------- File Browser
                     +----------------- n8n (optional)
                     |
                    Samba
                     |
                1TB USB HDD
                     |
        +------------+------------+
        |            |            |
      photos       movies       shared
        |
     backups
```

## Important design rule

This machine is **not** a general-purpose high-performance server.

With 4GB RAM and 2 CPU threads:

- Add one service.
- Test it.
- Watch RAM and CPU.
- Then add the next service.

Do **not** install everything and hope it works.

---

# 2. Before touching anything: collect your current information

This section is intentionally detailed.

Log into the Ubuntu machine locally.

If you have a monitor and keyboard connected to the server:

1. Turn on the server.
2. Wait for Ubuntu to finish booting.
3. Open a Terminal if you have the GUI.
4. If Ubuntu is already terminal-only, just log in.

Run:

```bash
whoami
hostname
hostname -I
free -h
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS,MODEL,SERIAL
```

Write down the results somewhere.

You especially want to know:

- Linux username
- Hostname
- Current IP address
- RAM
- Internal disk
- External 1TB disk
- Whether a disk already contains data

### Do not format anything yet.

The most dangerous command in the whole guide is a filesystem-formatting command such as:

```bash
sudo mkfs.ext4 /dev/sdb1
```

That can erase the existing contents of that partition.

We will identify the correct disk first.

---

# 3. Check whether Ubuntu is Desktop or Server

Run:

```bash
systemctl get-default
```

### Result A

If you see:

```text
graphical.target
```

Ubuntu normally boots to a graphical desktop.

### Result B

If you see:

```text
multi-user.target
```

Ubuntu is operating in a non-graphical/server-style boot target.

Also run:

```bash
echo "$XDG_CURRENT_DESKTOP"
```

Possible results may include:

```text
ubuntu:GNOME
```

or nothing.

Also run:

```bash
free -h
```

Look at the `available` memory.

### Why we care

A GUI consumes memory that you do not need for a headless server.

With only 4GB RAM, the goal is to eventually have the machine boot into:

```text
multi-user.target
```

---

# 4. Check whether the internal disk is SSD or HDD

Run:

```bash
lsblk -d -o NAME,SIZE,ROTA,MODEL
```

Example:

```text
NAME   SIZE ROTA MODEL
sda    223G    0 Samsung SSD
sdb    931G    1 External USB HDD
```

Interpretation:

- `ROTA=0` → normally SSD
- `ROTA=1` → spinning HDD

Your internal disk being an HDD is not a showstopper, but swap will be slower.

---

# 5. Decide whether you need to reinstall Ubuntu

## Recommended approach

If your current Ubuntu installation is already stable, **do not reinstall only because this guide says “Ubuntu Server.”**

You can turn the existing installation into a headless server.

A clean Ubuntu Server LTS installation is also valid, but it is more disruptive because you have to:

- back up anything important;
- boot from USB installer;
- partition the internal disk;
- reinstall;
- recreate users;
- reinstall drivers;
- redo networking.

For a beginner, keeping the working installation is normally safer.

---

# 6. Make Desktop Ubuntu boot without the GUI

Skip this section if your machine already boots to a terminal.

First check whether GDM exists:

```bash
systemctl status gdm3 --no-pager
```

If that says the service does not exist, check LightDM:

```bash
systemctl status lightdm --no-pager
```

Another useful command:

```bash
systemctl list-unit-files | grep -E 'gdm3|lightdm|sddm'
```

## If you use GNOME/GDM

Run:

```bash
sudo systemctl set-default multi-user.target
sudo systemctl disable gdm3
```

## If you use LightDM

Run:

```bash
sudo systemctl set-default multi-user.target
sudo systemctl disable lightdm
```

Then reboot:

```bash
sudo reboot
```

The machine should come back without loading the graphical desktop.

Check:

```bash
systemctl get-default
```

You want:

```text
multi-user.target
```

## Important

Do not uninstall the desktop yet.

The goal is simply:

**keep the software installed, but stop starting the GUI automatically.**

That makes recovery easier if you need to plug a monitor back in.

---

# 7. Find your server's local IP address

Run:

```bash
hostname -I
```

Example:

```text
192.168.1.50
```

That IP is what other devices on your home network use to reach the server.

Also run:

```bash
ip route
```

You should see a default route similar to:

```text
default via 192.168.1.1 dev enp3s0
```

Here:

- `192.168.1.1` is probably your router.
- `enp3s0` is your network interface.

Find all interfaces:

```bash
ip -br addr
```

Example:

```text
lo       UNKNOWN 127.0.0.1/8
enp3s0   UP      192.168.1.50/24
```

---

# 8. Find the server's MAC address

The MAC address is useful for creating a DHCP reservation in your router.

Run:

```bash
ip -br link
```

Or:

```bash
ip link
```

Example:

```text
enp3s0 UP ...
```

Then:

```bash
cat /sys/class/net/enp3s0/address
```

Example:

```text
a4:bb:6d:12:34:56
```

Your interface may be named differently.

Use the interface shown by:

```bash
ip -br addr
```

---

# 9. Give the server a stable IP address

This is better than manually hard-coding a static IP on Ubuntu.

Use your **router's DHCP reservation**.

That means:

> The router will always give this server the same local IP.

## Step 1 — open your router's admin page

From another device on your home network, open a browser.

Typical router addresses are:

```text
192.168.0.1
192.168.1.1
10.0.0.1
```

Do not guess if you can avoid it.

On Ubuntu, run:

```bash
ip route | grep default
```

If you get:

```text
default via 192.168.1.1 dev enp3s0
```

then your router is probably:

```text
192.168.1.1
```

Open that address in the browser.

## Step 2 — log into the router

Use the router admin username/password.

This information is usually:

- printed on the router;
- provided by your ISP;
- visible in the router app;
- or shown in the installation paperwork.

Do not use your normal Wi-Fi password unless your router documentation says they are the same.

## Step 3 — find DHCP settings

Different routers use different names.

Look for one of:

- LAN
- Network
- DHCP
- DHCP Server
- Address Reservation
- Static Lease
- IP Reservation
- Attached Devices
- Connected Devices

## Step 4 — find your server

Look for:

- hostname;
- MAC address;
- current IP;
- manufacturer.

Match the MAC address you recorded earlier.

## Step 5 — reserve the address

For example:

```text
Device: A6-server
MAC: a4:bb:6d:12:34:56
Reserved IP: 192.168.1.50
```

Save/apply.

## Step 6 — verify

On Ubuntu:

```bash
hostname -I
```

It should still show the same IP.

Then from your laptop:

```bash
ping 192.168.1.50
```

Replace the IP with your actual server IP.

Stop pinging with:

```text
Ctrl+C
```

---

# 10. Update Ubuntu before installing anything else

Run:

```bash
sudo apt update
```

Then:

```bash
sudo apt full-upgrade -y
```

Then reboot:

```bash
sudo reboot
```

After the reboot, log back in.

Check:

```bash
uptime
```

---

# 11. Install basic administration tools

Run:

```bash
sudo apt install -y \
  curl \
  wget \
  git \
  vim \
  nano \
  htop \
  ncdu \
  unzip \
  ca-certificates \
  gnupg \
  lsb-release \
  smartmontools \
  rsync \
  jq
```

You do not need to memorize what all of these do.

A few useful ones:

- `htop` → CPU/RAM monitor
- `ncdu` → find what is consuming disk space
- `smartctl` → inspect disk health
- `rsync` → backups
- `nano` → beginner-friendly text editor

---

# 12. Enable automatic security updates

Install:

```bash
sudo apt install -y unattended-upgrades
```

Then configure:

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

When asked whether to enable automatic updates, choose:

```text
Yes
```

Check the service:

```bash
systemctl status unattended-upgrades --no-pager
```

---

# 13. Install and configure SSH

SSH lets you administer the server from another computer without keeping a monitor attached.

Install OpenSSH:

```bash
sudo apt install -y openssh-server
```

Check:

```bash
sudo systemctl status ssh --no-pager
```

You want:

```text
active (running)
```

Find the server IP again:

```bash
hostname -I
```

From another Linux/macOS machine:

```bash
ssh YOUR_USERNAME@SERVER_IP
```

Example:

```bash
ssh hrithik@192.168.1.50
```

From Windows PowerShell, the same command normally works:

```powershell
ssh YOUR_USERNAME@SERVER_IP
```

---

# 14. Windows: create an SSH key

Do this on your Windows laptop/PC, not on the server.

Open:

```text
PowerShell
```

Run:

```powershell
ssh-keygen -t ed25519
```

When asked:

```text
Enter file in which to save the key:
```

Press Enter to accept the default:

```text
C:\Users\YOUR_NAME\.ssh\id_ed25519
```

When asked for a passphrase:

- using a passphrase is safer;
- do not reuse your normal account password.

Afterward, check:

```powershell
dir $env:USERPROFILE\.ssh
```

You should see:

```text
id_ed25519
id_ed25519.pub
```

The file ending in `.pub` is the public key.

**Never send the private key `id_ed25519` to anyone.**

---

# 15. Put the SSH public key on the server

From Windows PowerShell:

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh YOUR_USERNAME@SERVER_IP "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Replace:

- `YOUR_USERNAME`
- `SERVER_IP`

You may be asked for the server user's current password once.

Then test:

```powershell
ssh YOUR_USERNAME@SERVER_IP
```

It should authenticate using the key.

---

# 16. Only after key login works: disable SSH password authentication

This is an important safety rule:

**Do not disable password authentication first.**

First make sure key login works.

On the server:

```bash
sudo nano /etc/ssh/sshd_config
```

Find:

```text
PasswordAuthentication yes
```

Change it to:

```text
PasswordAuthentication no
```

Also check:

```text
PubkeyAuthentication yes
```

If the line is commented out, for example:

```text
#PubkeyAuthentication yes
```

you may change it to:

```text
PubkeyAuthentication yes
```

Save in nano:

1. `Ctrl+O`
2. Enter
3. `Ctrl+X`

Check the SSH configuration before restarting:

```bash
sudo sshd -t
```

### No output is good.

If there is an error:

**Do not restart SSH.**

Read the error and fix the configuration first.

If the test is clean:

```bash
sudo systemctl restart ssh
```

Open a **new** terminal on your laptop and test again:

```powershell
ssh YOUR_USERNAME@SERVER_IP
```

Keep your current SSH session open until the new connection works.

---

# 17. Install UFW firewall

Install:

```bash
sudo apt install -y ufw
```

Do not enable it blindly.

First inspect the current status:

```bash
sudo ufw status verbose
```

Set defaults:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow SSH from your LAN.

If your home network is:

```text
192.168.1.0/24
```

use:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
```

If your home network is:

```text
192.168.0.0/24
```

use:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 22 proto tcp
```

To discover your subnet:

```bash
ip -br addr
```

For example:

```text
192.168.1.50/24
```

means the subnet is normally:

```text
192.168.1.0/24
```

Now enable UFW:

```bash
sudo ufw enable
```

Then:

```bash
sudo ufw status numbered
```

## When you later install Samba

Samba is a host service, not a Docker-published port, so UFW rules matter.

For a typical LAN such as `192.168.1.0/24`, allow SMB from the LAN:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 445 proto tcp
```

If you need legacy NetBIOS discovery/older clients, you may also need:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 139 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 137 proto udp
sudo ufw allow from 192.168.1.0/24 to any port 138 proto udp
```

Only add these extra legacy rules if you actually need them.

If you want Samba access over Tailscale too, you can later allow trusted traffic arriving on the Tailscale interface:

```bash
sudo ufw allow in on tailscale0
```

That rule is broad for your tailnet, so use it only if you trust all devices/users that can join that tailnet.

---

# 18. Important Docker + UFW warning

Once Docker is installed, published container ports are handled by Docker's networking rules. Docker documents that published ports can bypass the normal UFW filtering path.

That means this:

```yaml
ports:
  - "8096:8096"
```

does **not** automatically mean:

> "UFW will block everyone except the subnet I allowed."

Treat Docker-published ports as network exposure that must be intentionally controlled.

For this home-server design:

- do not publish services to the public internet through the router;
- prefer LAN access and Tailscale;
- only publish ports that the service actually needs;
- later, if you need additional filtering around Docker, use Docker's documented `DOCKER-USER` firewall path rather than assuming UFW alone covers published containers.

Reference: Docker's official documentation on Docker/ufw firewall interaction.

---

# 19. Add swap

With only 4GB RAM, swap is useful as an emergency buffer.

Check existing swap first:

```bash
swapon --show
```

If nothing is listed, create a 4GB swap file.

## Step 1 — create it

```bash
sudo fallocate -l 4G /swapfile
```

## Step 2 — restrict permissions

```bash
sudo chmod 600 /swapfile
```

## Step 3 — mark it as swap

```bash
sudo mkswap /swapfile
```

## Step 4 — turn it on

```bash
sudo swapon /swapfile
```

## Step 5 — make it survive reboot

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Step 6 — confirm

```bash
swapon --show
free -h
```

You should now see approximately 4GB of swap.

## Optional tuning

Create:

```bash
sudo nano /etc/sysctl.d/99-server-memory.conf
```

Put:

```text
vm.swappiness=10
```

Save.

Apply:

```bash
sudo sysctl --system
```

Check:

```bash
sysctl vm.swappiness
```

---

# 20. Identify the 1TB USB drive safely

Plug in the external drive.

Run:

```bash
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS,MODEL,SERIAL
```

Example:

```text
NAME   SIZE FSTYPE TYPE MOUNTPOINTS MODEL
sda    223G        disk             SSD
└─sda1 223G ext4   part /           SSD
sdb    931G        disk             USB HDD
└─sdb1 931G ntfs   part /media/...  USB HDD
```

The device name may be:

```text
/dev/sdb
```

and the partition may be:

```text
/dev/sdb1
```

### Stop here if the disk contains files you need.

Do not run `mkfs`.

---

# 21. Inspect the filesystem before formatting

For example:

```bash
sudo blkid /dev/sdb1
```

If the result says:

```text
TYPE="ext4"
```

the drive is already ext4.

If it says:

```text
TYPE="ntfs"
```

it is NTFS.

If it says:

```text
TYPE="exfat"
```

it is exFAT.

You can decide later whether to keep the existing filesystem.

---

# 22. Optional: check the disk's health

Install SMART tools:

```bash
sudo apt install -y smartmontools
```

First inspect the drive:

```bash
sudo smartctl -a /dev/sdb
```

Some USB-to-SATA bridges do not expose SMART normally.

If that command says SMART is unavailable, do not assume the disk is dead; the USB adapter may simply not pass SMART data through.

---

# 23. Formatting the 1TB drive — ONLY if it contains nothing important

This is the destructive section.

You should only do this if:

- you confirmed `/dev/sdb1` is the correct partition;
- you copied any important files elsewhere;
- you are intentionally erasing the existing filesystem.

Format:

```bash
sudo mkfs.ext4 -L storage /dev/sdb1
```

Wait for it to finish.

Then check:

```bash
sudo blkid /dev/sdb1
```

You should see:

```text
TYPE="ext4"
LABEL="storage"
UUID="..."
```

Copy the UUID.

---

# 24. Create the storage mount point

Run:

```bash
sudo mkdir -p /mnt/storage
```

Test a temporary mount:

```bash
sudo mount /dev/sdb1 /mnt/storage
```

Check:

```bash
df -h /mnt/storage
```

You should see roughly 900GB usable space.

---

# 25. Mount by UUID so the drive doesn't break after reboot

Get UUID:

```bash
sudo blkid /dev/sdb1
```

Example:

```text
/dev/sdb1: UUID="1234-abcd-5678" TYPE="ext4" LABEL="storage"
```

Open fstab:

```bash
sudo nano /etc/fstab
```

Add one line at the bottom:

```text
UUID=1234-abcd-5678 /mnt/storage ext4 defaults,nofail 0 2
```

Replace the UUID with your actual one.

Save:

- `Ctrl+O`
- Enter
- `Ctrl+X`

Now test the configuration:

```bash
sudo umount /mnt/storage
sudo mount -a
```

Then:

```bash
df -h /mnt/storage
```

### Why test `mount -a`?

Because a typo in `/etc/fstab` can cause boot problems.

Testing it immediately is much safer than discovering the mistake after reboot.

---

# 26. Create the server directory structure

Run:

```bash
sudo mkdir -p /mnt/storage/photos
sudo mkdir -p /mnt/storage/movies
sudo mkdir -p /mnt/storage/shared
sudo mkdir -p /mnt/storage/backups
sudo mkdir -p /mnt/storage/appdata
sudo mkdir -p /mnt/storage/appdata/syncthing
sudo mkdir -p /mnt/storage/appdata/jellyfin
sudo mkdir -p /mnt/storage/appdata/filebrowser
sudo mkdir -p /mnt/storage/appdata/uptime-kuma
sudo mkdir -p /mnt/storage/appdata/pihole
sudo mkdir -p /mnt/storage/appdata/n8n
```

Check:

```bash
find /mnt/storage -maxdepth 2 -type d | sort
```

You should see the directories.

---

# 27. Understand who owns the storage

Find your Linux user's numeric UID/GID:

```bash
id
```

Example:

```text
uid=1000(hrithik) gid=1000(hrithik) groups=...
```

The official Syncthing container defaults to UID/GID 1000 and can be changed with `PUID`/`PGID`, so keeping a consistent 1000/1000 ownership model can simplify permissions.

Check current storage ownership:

```bash
ls -ld /mnt/storage
```

A simple starting model:

```bash
sudo chown -R YOUR_USERNAME:YOUR_USERNAME /mnt/storage
```

Replace `YOUR_USERNAME`.

Do not run that command on the wrong path.

Confirm:

```bash
ls -ld /mnt/storage /mnt/storage/photos /mnt/storage/movies /mnt/storage/shared
```

---

# 28. Install Docker the proper way

The original guide used:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

That convenience script is useful for testing/development, but Docker's current Ubuntu documentation recommends installing Docker Engine from Docker's apt repository for a normal managed installation.

## Step 1 — remove conflicting older packages if present

```bash
sudo apt remove -y \
  docker.io \
  docker-compose \
  docker-doc \
  podman-docker \
  containerd \
  runc
```

If some packages were not installed, apt may simply report that.

## Step 2 — update apt

```bash
sudo apt update
```

## Step 3 — install prerequisites

```bash
sudo apt install -y ca-certificates curl
```

## Step 4 — create the keyring directory

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

## Step 5 — download Docker's signing key

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

## Step 6 — make the key readable

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

## Step 7 — add Docker's repository

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

## Step 8 — update package lists

```bash
sudo apt update
```

## Step 9 — install Docker Engine and Compose

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

## Step 10 — verify Docker

```bash
sudo systemctl status docker --no-pager
```

You want:

```text
active (running)
```

## Step 11 — run the official test container

```bash
sudo docker run hello-world
```

If the final output says the installation appears to be working correctly, Docker is alive.

---

# 29. Let your normal user run Docker

Run:

```bash
sudo usermod -aG docker $USER
```

You must start a new login session for the group membership to refresh.

For an SSH session:

1. Exit:

```bash
exit
```

2. SSH back in:

```bash
ssh YOUR_USERNAME@SERVER_IP
```

3. Test:

```bash
docker ps
```

You should not get:

```text
permission denied
```

### Security note

Membership in the `docker` group is effectively highly privileged because Docker can start containers with host-level access.

Only add trusted users to this group.

---

# 30. Verify Docker Compose

Run:

```bash
docker compose version
```

You should get a version.

Notice the current command is:

```bash
docker compose
```

not necessarily:

```bash
docker-compose
```

---

# 31. Create one directory for each application

Create a working directory:

```bash
mkdir -p ~/server-compose
```

Then:

```bash
cd ~/server-compose
```

We will keep individual stacks separate:

```text
~/server-compose/
├── syncthing/
├── filebrowser/
├── jellyfin/
├── uptime-kuma/
├── pihole/
└── n8n/
```

Create them:

```bash
mkdir -p ~/server-compose/syncthing
mkdir -p ~/server-compose/filebrowser
mkdir -p ~/server-compose/jellyfin
mkdir -p ~/server-compose/uptime-kuma
mkdir -p ~/server-compose/pihole
mkdir -p ~/server-compose/n8n
```

---

# 32. Container resource limits

This machine only has 4GB RAM.

A container memory limit should be treated as a safety boundary.

Example:

```yaml
mem_limit: 256m
```

This means Docker will restrict the container to approximately 256MB RAM.

Do not copy limits blindly.

A service that genuinely needs more memory will need a larger limit.

---

# 33. Docker log rotation

Without log rotation, containers can eventually fill your disk.

Create:

```bash
sudo nano /etc/docker/daemon.json
```

Put:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Save.

Validate JSON:

```bash
python3 -m json.tool /etc/docker/daemon.json
```

If it prints formatted JSON, the syntax is valid.

Then restart Docker:

```bash
sudo systemctl restart docker
```

Check:

```bash
sudo systemctl status docker --no-pager
```

---

# 34. Always use restart policies

For long-running services use:

```yaml
restart: unless-stopped
```

This means:

- start after Docker comes back;
- recover after a container crash;
- do not restart if you intentionally stopped it.

---

# 35. Service 1 — Samba family file share

Samba is for normal network-file-sharing behavior.

It is useful when a family member wants:

```text
\\A6-SERVER\Family Share
```

from Windows.

## Step 1 — install Samba

```bash
sudo apt install -y samba
```

## Step 2 — make the shared directory

Already created:

```text
/mnt/storage/shared
```

Verify:

```bash
ls -ld /mnt/storage/shared
```

## Step 3 — create a Samba password for your Linux user

Run:

```bash
sudo smbpasswd -a YOUR_USERNAME
```

Enter a Samba password.

This does not have to be the same as your Linux password.

## Step 4 — back up the Samba configuration

Before editing:

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
```

## Step 5 — edit Samba configuration

Open:

```bash
sudo nano /etc/samba/smb.conf
```

Scroll to the bottom.

Add:

```ini
[Family Share]
   path = /mnt/storage/shared
   browseable = yes
   writable = yes
   guest ok = no
   read only = no
```

Save:

- `Ctrl+O`
- Enter
- `Ctrl+X`

## Step 6 — validate the configuration

Run:

```bash
testparm
```

If it reports no critical syntax problems, continue.

## Step 7 — restart Samba

```bash
sudo systemctl restart smbd
```

Check:

```bash
sudo systemctl status smbd --no-pager
```

---

# 36. Test Samba from Windows

On Windows:

1. Press `Win+R`.
2. Type:

```text
\\SERVER_IP\Family Share
```

Example:

```text
\\192.168.1.50\Family Share
```

3. Press Enter.
4. When prompted, use your Samba username/password.

Create a test file:

```text
samba-test.txt
```

Then verify on Ubuntu:

```bash
ls -l /mnt/storage/shared
```

You should see the test file.

---

# 37. Optional: map the Samba share as a Windows drive

On Windows:

1. Open File Explorer.
2. Right-click **This PC**.
3. Select **Map network drive**.
4. Choose a drive letter, for example `Z:`.
5. Enter:

```text
\\192.168.1.50\Family Share
```

6. Enable reconnect if desired.
7. Enter Samba credentials.

Now the server appears as a normal drive.

---

# 38. Service 2 — Syncthing

Syncthing is for automatic synchronization.

The desired flow is:

```text
Phone Camera folder
        |
        | Syncthing
        v
/mnt/storage/photos/PersonName/
```

Important distinction:

**Syncthing is synchronization, not a complete backup strategy.**

If a file is deleted and that deletion synchronizes, the server copy may also disappear depending on the folder configuration.

That is why we also create separate backups later.

---

# 39. Create a Syncthing Compose file

Go to:

```bash
cd ~/server-compose/syncthing
```

Create:

```bash
nano compose.yml
```

Paste:

```yaml
services:
  syncthing:
    image: syncthing/syncthing:latest
    container_name: syncthing
    hostname: a6-syncthing
    environment:
      - PUID=1000
      - PGID=1000
      - STGUIADDRESS=
    volumes:
      - /mnt/storage/appdata/syncthing:/var/syncthing
      - /mnt/storage/photos:/photos
    network_mode: host
    restart: unless-stopped
    mem_limit: 256m
```

Save.

The official Syncthing Docker documentation recommends host networking because it allows better LAN discovery behavior.

---

# 40. Start Syncthing

From:

```bash
cd ~/server-compose/syncthing
```

run:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs --tail=100
```

Follow live logs:

```bash
docker compose logs -f
```

Press:

```text
Ctrl+C
```

to stop following logs.

---

# 41. Open Syncthing web interface

On another machine in your home network, open:

```text
http://SERVER_IP:8384
```

Example:

```text
http://192.168.1.50:8384
```

## Important

Do not leave the Syncthing GUI publicly exposed.

Because we are using Tailscale later, remote access will be through the private tailnet.

---

# 42. Secure the Syncthing GUI

In the Syncthing web interface:

1. Open **Actions**.
2. Open **Settings**.
3. Go to **GUI**.
4. Set a GUI username.
5. Set a strong GUI password.
6. Save.

Do this before giving access to anyone else.

---

# 43. Connect a phone to Syncthing

Install the Syncthing client appropriate for your phone platform.

Open Syncthing on the phone.

On the server:

1. Open Syncthing.
2. Find **Actions → Show ID**.
3. Copy the server device ID or use the QR code.

On the phone:

1. Add the server as a remote device.
2. Enter/scan the device ID.
3. Give it a friendly name such as `Home A6 Server`.

Back on the server, accept the new device.

The two devices are now paired.

---

# 44. Configure the phone photo folder

Create a server-side directory for each person.

For example:

```bash
sudo mkdir -p /mnt/storage/photos/hrithik
sudo mkdir -p /mnt/storage/photos/mom
sudo mkdir -p /mnt/storage/photos/dad
```

Adjust names to your family.

On the phone:

1. Add the Camera/DCIM folder to Syncthing.
2. Choose the server as the shared device.
3. On the server, accept the folder.
4. Choose the server path:

```text
/photos/hrithik
```

or the appropriate person.

Wait for synchronization.

---

# 45. Verify the photo sync

On the phone, take one new test photo.

Wait for Syncthing to report synchronization completed.

On Ubuntu:

```bash
find /mnt/storage/photos -type f | tail
```

Or:

```bash
ls -lah /mnt/storage/photos/hrithik
```

You should see the test photo.

---

# 46. Service 3 — File Browser

File Browser gives less technical family members a web page for uploading/downloading files.

We will use:

```text
/mnt/storage/shared
```

as its root directory.

---

# 47. Create the File Browser stack

Go to:

```bash
cd ~/server-compose/filebrowser
```

Create:

```bash
nano compose.yml
```

Paste:

```yaml
services:
  filebrowser:
    image: filebrowser/filebrowser:latest
    container_name: filebrowser
    volumes:
      - /mnt/storage/shared:/srv
      - /mnt/storage/appdata/filebrowser:/database
    ports:
      - "8081:80"
    restart: unless-stopped
    mem_limit: 128m
```

Save.

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs --tail=100
```

---

# 48. Open File Browser

From a browser:

```text
http://SERVER_IP:8081
```

Example:

```text
http://192.168.1.50:8081
```

On first startup, inspect the container logs because File Browser's initial bootstrap credentials may be printed there.

Run:

```bash
docker logs filebrowser
```

Save the initial credentials securely and change the password immediately.

---

# 49. Test File Browser

1. Log in.
2. Create a folder.
3. Upload a small test file.
4. Download it again.
5. Delete the test file.

On Ubuntu verify:

```bash
find /mnt/storage/shared -maxdepth 2 -type f
```

---

# 50. Service 4 — Jellyfin

Jellyfin is for your family movie/TV library.

The most important rule on this hardware is:

**Prefer Direct Play over transcoding.**

With only two CPU threads, live CPU transcoding can overwhelm the machine quickly.

---

# 51. Prepare the Jellyfin directories

Check:

```bash
ls -ld /mnt/storage/movies
ls -ld /mnt/storage/appdata/jellyfin
```

You can optionally make separate libraries:

```bash
sudo mkdir -p /mnt/storage/movies/Movies
sudo mkdir -p /mnt/storage/movies/TV
```

---

# 52. Create Jellyfin Compose file

Go to:

```bash
cd ~/server-compose/jellyfin
```

Create:

```bash
nano compose.yml
```

Paste:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: "1000:1000"
    volumes:
      - /mnt/storage/appdata/jellyfin:/config
      - /mnt/storage/movies:/media:ro
    ports:
      - "8096:8096"
    restart: unless-stopped
    mem_limit: 768m
```

Save.

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs --tail=100
```

---

# 53. Open Jellyfin

Open:

```text
http://SERVER_IP:8096
```

Example:

```text
http://192.168.1.50:8096
```

Complete the setup wizard.

When choosing the media library:

For movies:

```text
/media/Movies
```

For TV:

```text
/media/TV
```

The container sees:

```text
/mnt/storage/movies
```

as:

```text
/media
```

because of the Docker mount.

---

# 54. Add one small test movie first

Do not immediately copy your entire movie collection.

First place one known-good video in:

```text
/mnt/storage/movies/Movies/
```

Then in Jellyfin:

1. Refresh the library.
2. Open the movie.
3. Play it.
4. Check the playback information.

You want to see:

```text
Direct Play
```

rather than:

```text
Transcoding
```

---

# 55. How to test CPU usage during Jellyfin playback

While a video is playing, SSH into the server and run:

```bash
htop
```

Also run:

```bash
docker stats
```

Watch:

- CPU percentage
- RAM usage
- Jellyfin container memory
- system load

If a simple video causes CPU to sit extremely high, make Direct Play the priority.

---

# 56. Jellyfin hardware acceleration: treat it as optional

First find the GPU:

```bash
lspci -nn | grep -Ei "3d|display|vga"
```

Check whether a DRM device exists:

```bash
ls -lah /dev/dri
```

If `/dev/dri` does not exist, stop here.

Do not force hardware acceleration.

If it exists, you can investigate VA-API.

For modern supported AMD GPUs, Jellyfin documents VA-API as the preferred Linux acceleration interface.

Older AMD hardware can have limitations, so on this particular machine the first goal should remain **Direct Play**.

---

# 57. Optional Jellyfin VA-API setup

Only do this if:

```bash
ls -lah /dev/dri
```

shows something such as:

```text
card0
renderD128
```

Find the render group:

```bash
getent group render
```

Example:

```text
render:x:122:
```

Here the group ID is:

```text
122
```

Edit Jellyfin Compose:

```bash
nano ~/server-compose/jellyfin/compose.yml
```

Add:

```yaml
    group_add:
      - "122"
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
```

Use the actual `render` group ID from your machine.

Then:

```bash
cd ~/server-compose/jellyfin
docker compose up -d
```

Inside Jellyfin:

1. Open **Dashboard**.
2. Open **Playback** / **Transcoding**.
3. Select VA-API where supported.
4. Save.
5. Test a transcode deliberately.

Do not assume acceleration is working just because the option exists.

---

# 58. Service 5 — Tailscale

Tailscale is what lets you reach the server from outside your home without port-forwarding every service to the internet.

The original design uses:

```text
Phone/laptop
     |
 mobile data / hotel Wi-Fi
     |
  Tailscale
     |
  A6 server
```

---

# 59. Install Tailscale on Ubuntu

Run:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Then:

```bash
sudo tailscale up
```

Tailscale will show an authentication URL.

Copy/open that URL in a browser.

Sign into your Tailscale account.

After authentication, check:

```bash
tailscale status
```

Also run:

```bash
tailscale ip
```

You should get a Tailscale IP, normally in the 100.x.x.x range.

---

# 60. Install Tailscale on family devices

Each family member should normally use their own account/device identity rather than sharing one password.

Install Tailscale on:

- Android phones
- iPhones
- laptops
- tablets
- other machines that need remote access

Sign each device into the appropriate tailnet account.

From a family phone outside home, enable Tailscale.

Then open:

```text
http://TAILSCALE_IP:8096
```

for Jellyfin, or:

```text
http://TAILSCALE_IP:8081
```

for File Browser.

---

# 61. Test Tailscale from outside your home

Do not only test while connected to home Wi-Fi.

Use a phone:

1. Turn off Wi-Fi.
2. Leave mobile data on.
3. Enable Tailscale.
4. Open:

```text
http://TAILSCALE_IP:8096
```

If Jellyfin loads, your remote path is working.

This is one of the most important checkpoints.

---

# 62. Do NOT port-forward the services by default

Do not create router port forwards like:

```text
8096 -> SERVER
8384 -> SERVER
8081 -> SERVER
3001 -> SERVER
5678 -> SERVER
```

unless you deliberately understand the security implications.

For this family-server design, use:

- LAN access when at home;
- Tailscale when away.

---

# 63. Service 6 — Pi-hole

Pi-hole is a DNS-level network ad blocker.

## Important warning

Pi-hole becomes infrastructure for your whole home network.

If Pi-hole goes down and your router is configured to use only Pi-hole for DNS, family members may suddenly have DNS failures.

Therefore:

**Install this only after the core services are stable.**

---

# 64. Check whether port 53 is already in use

Before Pi-hole:

```bash
sudo ss -lntup | grep ':53 '
```

Also:

```bash
sudo ss -lunp | grep ':53 '
```

If something is already listening on port 53, you must understand what it is before continuing.

Common examples involve local DNS stub resolvers.

Do not randomly kill DNS services.

---

# 65. Create Pi-hole stack

Go to:

```bash
cd ~/server-compose/pihole
```

Create:

```bash
nano compose.yml
```

Use a LAN-safe port mapping:

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    environment:
      TZ: Asia/Kolkata
    volumes:
      - /mnt/storage/appdata/pihole/etc-pihole:/etc/pihole
      - /mnt/storage/appdata/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8082:80/tcp"
    restart: unless-stopped
    mem_limit: 192m
```

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs --tail=100
```

---

# 66. Open Pi-hole

Open:

```text
http://SERVER_IP:8082
```

Example:

```text
http://192.168.1.50:8082
```

Complete the initial setup and set a strong admin password.

---

# 67. Change router DNS only after Pi-hole is working

Do not change the router DNS first.

First verify Pi-hole itself works.

On the server:

```bash
docker exec -it pihole pihole status
```

Then from another device, temporarily configure its DNS manually to:

```text
SERVER_IP
```

Example:

```text
192.168.1.50
```

Test browsing.

If it works, then you can consider changing the router's DHCP/DNS settings so clients automatically use Pi-hole.

---

# 68. How to undo Pi-hole if the network breaks

Know the recovery path before changing DNS.

On the router, change DNS back to:

- automatic;
- ISP DNS;
- or another trusted DNS provider.

The exact menu varies by router.

You should not be locked out of the network just because the server rebooted.

---

# 69. Service 7 — Uptime Kuma

Uptime Kuma is a monitoring dashboard.

It can monitor:

- Jellyfin
- File Browser
- Syncthing
- Pi-hole
- Tailscale endpoints
- other HTTP/TCP services

---

# 70. Create Uptime Kuma stack

Go to:

```bash
cd ~/server-compose/uptime-kuma
```

Create:

```bash
nano compose.yml
```

Paste:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    volumes:
      - /mnt/storage/appdata/uptime-kuma:/app/data
    ports:
      - "3001:3001"
    restart: unless-stopped
    mem_limit: 192m
```

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Open:

```text
http://SERVER_IP:3001
```

Create the admin user.

---

# 71. Add your first Uptime Kuma monitor

Inside Uptime Kuma:

1. Select **Add New Monitor**.
2. Monitor type: **HTTP(s)**.
3. Friendly name:

```text
Jellyfin
```

4. URL:

```text
http://SERVER_IP:8096
```

5. Save.

Then create similar monitors for:

```text
File Browser
Syncthing GUI
Pi-hole
```

---

# 72. Service 8 — Dashboard/Homepage

A dashboard is useful because family members should not have to memorize:

```text
8096
8081
8384
3001
8082
5678
```

For a 4GB server, keep the dashboard simple.

Start with bookmarks if necessary.

If you later install Homepage/Homarr, give it only a small memory budget and avoid unnecessary widgets that constantly poll services.

---

# 73. Service 9 — n8n automation

n8n is optional.

Only install it after the core stack is stable.

Good uses on a small machine:

- webhook receiver;
- scheduled tasks;
- Telegram bot;
- API orchestration;
- calling cloud AI;
- notifying the family about changes.

Bad use for this machine:

- running a local large language model.

---

# 74. Create n8n stack

Go to:

```bash
cd ~/server-compose/n8n
```

Create:

```bash
nano compose.yml
```

Paste:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    volumes:
      - /mnt/storage/appdata/n8n:/home/node/.n8n
    ports:
      - "5678:5678"
    restart: unless-stopped
    mem_limit: 384m
```

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Open:

```text
http://SERVER_IP:5678
```

Follow the setup wizard.

Do not expose n8n directly to the public internet without deliberately securing it.

---

# 75. Do not run a local LLM on this machine

With:

- 2 CPU threads
- 4GB RAM

a local LLM is not a sensible first target.

A much better architecture is:

```text
n8n
  |
  +----> cloud AI API
```

rather than:

```text
n8n
  |
  +----> local model consuming most server RAM
```

That gives you useful automation without turning the server into a heater.

---

# 76. Example family automation ideas

Once n8n is stable, you could build:

### Idea A — daily backup report

Every evening:

```text
"What was added to family photos today?"
```

n8n checks the folder and sends a message.

### Idea B — movie lookup bot

A family member asks:

```text
Do we have any space movies?
```

The workflow searches metadata/files and replies.

### Idea C — backup warning

If free disk space falls below a threshold:

```text
"Storage is below 15%; please clean or expand it."
```

This is much more valuable than running an AI model locally.

---

# 77. Check overall server health

After installing a few services, run:

```bash
htop
```

In another terminal:

```bash
docker stats
```

Also:

```bash
free -h
```

And:

```bash
df -h
```

You want to keep an eye on:

- RAM
- swap
- CPU
- disk usage

---

# 78. Watch CPU load

Run:

```bash
uptime
```

Example:

```text
load average: 0.25, 0.30, 0.28
```

On a 2-thread system:

- short spikes are normal;
- sustained very high load means something is consuming the machine.

Use:

```bash
htop
```

to find the process.

---

# 79. See which Docker container is using RAM

Run:

```bash
docker stats --no-stream
```

Example:

```text
NAME          CPU %   MEM USAGE / LIMIT
jellyfin      ...     ...
syncthing     ...     ...
pihole        ...     ...
```

Look for a container sitting near its memory limit.

---

# 80. See which directories consume disk space

Run:

```bash
sudo du -h /mnt/storage --max-depth=1 | sort -h
```

For the whole machine:

```bash
sudo du -h / --max-depth=1 2>/dev/null | sort -h
```

For interactive disk inspection:

```bash
sudo ncdu /
```

---

# 81. Check running containers

Run:

```bash
docker ps
```

To include stopped containers:

```bash
docker ps -a
```

---

# 82. Check Docker disk usage

Run:

```bash
docker system df
```

Do not blindly run:

```bash
docker system prune -a
```

on a production-ish server.

Pruning unused images is safe only when you understand what is actually unused.

---

# 83. Check container logs

For Jellyfin:

```bash
docker logs jellyfin --tail 100
```

For Syncthing:

```bash
docker logs syncthing --tail 100
```

For File Browser:

```bash
docker logs filebrowser --tail 100
```

For Pi-hole:

```bash
docker logs pihole --tail 100
```

---

# 84. Follow logs live

Example:

```bash
docker logs -f jellyfin
```

Press:

```text
Ctrl+C
```

to stop following.

The container keeps running.

---

# 85. Start / stop / restart a service

From the service directory:

```bash
docker compose stop
```

Start again:

```bash
docker compose start
```

Restart:

```bash
docker compose restart
```

Update the stack configuration and recreate:

```bash
docker compose up -d
```

---

# 86. Stop one container directly

For example:

```bash
docker stop jellyfin
```

Start:

```bash
docker start jellyfin
```

---

# 87. Check whether a port is already occupied

If a service fails with something like:

```text
port is already allocated
```

check:

```bash
sudo ss -lntup | grep ':8096'
```

Replace `8096` with the port you are investigating.

Also:

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

This often reveals the conflict immediately.

---

# 88. Basic troubleshooting decision tree

When a web service does not load:

## Step 1 — Is the container running?

```bash
docker ps
```

If it is not listed:

```bash
docker ps -a
```

## Step 2 — Read logs

```bash
docker logs CONTAINER_NAME --tail 100
```

## Step 3 — Does the port exist?

```bash
sudo ss -lntup | grep ':PORT'
```

## Step 4 — Is the server reachable?

From another machine:

```bash
ping SERVER_IP
```

## Step 5 — Can the server reach its own port?

On the server:

```bash
curl -I http://127.0.0.1:PORT
```

## Step 6 — Is Docker healthy?

```bash
sudo systemctl status docker --no-pager
```

This sequence prevents random guessing.

---

# 89. Common permission problem

Example error:

```text
Permission denied
```

First inspect the directory:

```bash
ls -ld /mnt/storage/photos
```

Then inspect the contents:

```bash
ls -lah /mnt/storage/photos
```

Check the container's expected UID/GID.

For Syncthing:

```text
1000:1000
```

is the default.

Use:

```bash
id YOUR_USERNAME
```

to compare.

Do not solve every permission problem with:

```bash
chmod -R 777
```

Avoid that.

It gives every local user and service unnecessarily broad access.

---

# 90. Common mount problem after reboot

If `/mnt/storage` is empty after reboot:

Run:

```bash
findmnt /mnt/storage
```

Then:

```bash
cat /etc/fstab
```

And:

```bash
sudo mount -a
```

If `mount -a` reports an error, fix `/etc/fstab`.

Also check:

```bash
lsblk -f
```

to confirm the partition UUID.

---

# 91. Common USB disk problem

If the external disk disappears:

Run:

```bash
lsblk -o NAME,SIZE,FSTYPE,MODEL,SERIAL,MOUNTPOINTS
```

Then:

```bash
dmesg | tail -n 50
```

Look for USB disconnects or filesystem errors.

Physically check:

- USB cable
- USB port
- external power adapter if the enclosure has one

A family photo server should ideally use a reliable external drive/enclosure rather than a loose, constantly replugged portable disk.

---

# 92. Backup plan: understand the difference between sync and backup

This is extremely important.

### Syncthing

```text
Phone <----sync----> Server
```

Useful for synchronization.

### Backup

```text
Server ----copy----> separate backup device
```

Useful when the original or server is lost.

You need both.

---

# 93. Create a monthly cold backup

Plug in a backup pendrive or other backup disk.

Find it:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,MODEL
```

Mount it.

For example, if it becomes:

```text
/media/YOUR_USERNAME/BACKUP_DRIVE
```

create:

```bash
mkdir -p /media/YOUR_USERNAME/BACKUP_DRIVE/photos
```

Then copy:

```bash
rsync -av --delete \
  /mnt/storage/photos/ \
  /media/YOUR_USERNAME/BACKUP_DRIVE/photos/
```

### Important

`--delete` means files removed from the source can also be removed from the backup.

That may be desirable for a mirror but is not the same as versioned backup.

For irreplaceable photos, consider keeping historical copies instead of only one mirror.

---

# 94. Safely eject the backup disk

Before unplugging:

```bash
sync
```

Then unmount it.

For example:

```bash
sudo umount /media/YOUR_USERNAME/BACKUP_DRIVE
```

Check:

```bash
findmnt
```

Make sure it is no longer mounted before physically unplugging.

---

# 95. Cloud backup for irreplaceable photos

The strongest simple design is:

```text
Phone
  |
  v
A6 server
  |
  +---- local backup disk
  |
  +---- cloud backup
```

That gives protection against:

- disk failure;
- accidental deletion;
- USB enclosure failure;
- theft;
- fire;
- power-related damage.

Cloud storage choice depends on what account/provider you already use and the amount of data.

---

# 96. Test your backup

A backup is not real until it has been restored successfully at least once.

Take one test photo.

Back it up.

Then simulate failure:

1. Delete the test photo from the primary server copy.
2. Restore it from the backup.
3. Open it.
4. Confirm it is intact.

Do this occasionally.

---

# 97. Create a simple server inventory

Create:

```bash
mkdir -p ~/server-notes
nano ~/server-notes/server-info.txt
```

Record:

```text
Hostname:
Ubuntu version:
Server IP:
Tailscale IP:
Network interface:
Router IP:
MAC address:
Storage device:
Storage UUID:
SSH username:
Docker version:
Compose version:
```

Do not record:

- passwords;
- private SSH keys;
- Tailscale authentication secrets.

---

# 98. Save copies of important configuration files

Back up:

```bash
/etc/fstab
/etc/ssh/sshd_config
/etc/samba/smb.conf
/etc/ufw/*
/etc/docker/daemon.json
```

For example:

```bash
mkdir -p ~/server-notes/config-backup
```

Then:

```bash
sudo cp /etc/fstab ~/server-notes/config-backup/
sudo cp /etc/ssh/sshd_config ~/server-notes/config-backup/
sudo cp /etc/samba/smb.conf ~/server-notes/config-backup/
sudo cp /etc/docker/daemon.json ~/server-notes/config-backup/
```

Be careful with permissions because some configuration files contain sensitive information.

---

# 99. Suggested installation order

Do not install everything on day one.

Use this order:

## Day 1 — base server

1. Identify hardware.
2. Decide whether to keep current Ubuntu.
3. Disable GUI if needed.
4. Record IP/MAC.
5. Create router DHCP reservation.
6. Update Ubuntu.
7. Install SSH.
8. Configure SSH key.
9. Disable SSH password login.
10. Configure UFW.
11. Add swap.
12. Identify and mount the 1TB drive.

## Day 2 — container foundation

13. Install Docker from Docker's official repository.
14. Verify Docker.
15. Install Compose.
16. Configure Docker log rotation.
17. Create storage/appdata directories.

## Day 3 — family file services

18. Install Samba.
19. Test Windows file access.
20. Install Syncthing.
21. Pair one phone.
22. Back up one test photo.
23. Install File Browser.
24. Upload/download one test file.

## Day 4 — media and remote access

25. Install Jellyfin.
26. Add one movie.
27. Confirm Direct Play.
28. Install Tailscale.
29. Test Jellyfin using mobile data.
30. Add family devices.

## Later

31. Pi-hole.
32. Uptime Kuma.
33. Dashboard.
34. n8n.
35. AI workflows.
36. Backup rotation.

---

# 100. Final target service table

| Service | Purpose | Default access | Always-on? |
|---|---|---:|---:|
| Samba | LAN file sharing | SMB/445 | Yes |
| Syncthing | Phone photo sync | GUI 8384 | Yes |
| File Browser | Simple web file access | 8081 | Optional |
| Jellyfin | Movies/TV | 8096 | Yes if used |
| Tailscale | Remote private access | VPN | Yes |
| Pi-hole | DNS/ad blocking | 8082 + DNS 53 | Yes if router uses it |
| Uptime Kuma | Monitoring | 3001 | Yes |
| Homepage/Homarr | Dashboard | varies | Optional |
| n8n | Automation | 5678 | Optional |

---

# 101. Commands worth memorizing

Check system:

```bash
htop
free -h
df -h
lsblk
```

Check network:

```bash
hostname -I
ip -br addr
ip route
```

Check Docker:

```bash
docker ps
docker ps -a
docker stats
docker logs CONTAINER_NAME
docker compose ps
```

Check services:

```bash
sudo systemctl status docker --no-pager
sudo systemctl status ssh --no-pager
sudo systemctl status smbd --no-pager
```

Check ports:

```bash
sudo ss -lntup
```

Check firewall:

```bash
sudo ufw status numbered
```

---

# 102. What NOT to do

Do not:

```bash
chmod -R 777 /
```

Do not:

```bash
docker system prune -a
```

without understanding what it removes.

Do not:

```bash
mkfs.ext4 /dev/sdX
```

until you have positively identified the disk.

Do not expose every service with router port forwarding.

Do not share the root account with family members.

Do not reuse one password everywhere.

Do not store private SSH keys on the server.

Do not assume Syncthing alone is a backup.

Do not assume Jellyfin hardware acceleration is working because the checkbox exists.

Do not install a local LLM just because the server can run Docker.

---

# 103. First-week acceptance test

At the end of the initial setup, verify every item below.

## Server

- [ ] Boots without GUI.
- [ ] Static/reserved LAN IP works.
- [ ] SSH key login works.
- [ ] Password SSH login is disabled.
- [ ] UFW is enabled.
- [ ] Swap is enabled.
- [ ] External HDD mounts after reboot.
- [ ] Docker starts automatically.

## Files

- [ ] Windows can open Samba share.
- [ ] A file can be uploaded.
- [ ] A file can be downloaded.
- [ ] Syncthing pairs with one phone.
- [ ] One test photo syncs.

## Jellyfin

- [ ] Jellyfin loads.
- [ ] One test movie is detected.
- [ ] One device can Direct Play.
- [ ] CPU remains reasonable during playback.

## Tailscale

- [ ] Server appears in the tailnet.
- [ ] Phone appears in the tailnet.
- [ ] Server is reachable using mobile data.
- [ ] Jellyfin works remotely through Tailscale.

## Monitoring

- [ ] Uptime Kuma loads.
- [ ] Jellyfin monitor is green.
- [ ] Syncthing monitor is green.
- [ ] File Browser monitor is green.

## Backup

- [ ] One external backup copy exists.
- [ ] Backup can be opened.
- [ ] At least one file has been restored successfully.

---

# 104. Recovery procedure if something goes badly wrong

If a new service breaks something:

## Step 1 — stop the new service

```bash
cd ~/server-compose/SERVICE_NAME
docker compose down
```

## Step 2 — check whether the host itself is healthy

```bash
free -h
df -h
uptime
sudo systemctl status docker --no-pager
```

## Step 3 — check mounts

```bash
findmnt /mnt/storage
```

## Step 4 — check networking

```bash
hostname -I
ip route
```

## Step 5 — restore the previous configuration if you changed a system file

Keep backups before editing:

```bash
sudo cp FILE FILE.backup
```

This is why the guide deliberately has you take backups before modifying:

- SSH
- Samba
- Docker
- fstab

---

# 105. The most important rule for this machine

Do not optimize for the number of services.

Optimize for:

```text
Reliable
   >
Simple
   >
Recoverable
   >
Then add more features
```

A stable server with:

```text
Samba
Syncthing
Jellyfin
Tailscale
Backups
```

is far more useful than a server with ten dashboards that crashes every few days.

---

# 106. Official references used for the technical steps

The detailed procedures in this guide should be checked against the current official documentation before making major changes:

- Docker Engine installation for Ubuntu — Docker Docs.
- Docker Compose plugin installation — Docker Docs.
- Docker networking and Docker/UFW interaction — Docker Docs.
- Tailscale Linux installation — Tailscale Docs.
- Syncthing Docker image and Docker networking — Syncthing documentation.
- Syncthing folder types — Syncthing documentation.
- Jellyfin container installation — Jellyfin Docs.
- Jellyfin hardware acceleration / AMD VA-API — Jellyfin Docs.
- Pi-hole Docker installation — Pi-hole Docs.
- File Browser Docker installation — File Browser documentation.

Because this is a live server, the exact UI labels and package versions can change over time. The commands in this guide intentionally use the service's documented installation paths rather than old package shortcuts.

---

# 107. Recommended final architecture

For this specific 2-core / 4GB machine, the practical end state is:

```text
                     INTERNET
                         |
                    TAILSCALE
                         |
                  +---------------+
                  | Ubuntu A6     |
                  | 4GB RAM       |
                  +---------------+
                   |  |  |  |  |
                   |  |  |  |  +---- Uptime Kuma
                   |  |  |  +------- Jellyfin
                   |  |  +---------- Syncthing
                   |  +------------- Samba
                   +---------------- Pi-hole
                   |
                 Docker
                   |
          +--------+--------+
          |        |        |
       FileBrowser n8n   future apps
          |
     /mnt/storage
          |
   +------+-------+--------+
   |      |       |        |
photos  movies  shared   backups
```

The priorities should remain:

1. **Data safety**
2. **Remote access security**
3. **Reliable file/photo service**
4. **Direct-play media**
5. **Monitoring**
6. **Automation**
7. **Experiments**

That order fits the hardware much better than trying to make the machine do everything at once.
