# Antigravity IDE in WSL (Arch Linux): Setup Guide

This guide provides a comprehensive walkthrough for setting up the Antigravity IDE within a custom Arch Linux WSL distribution (`vpn1`), featuring WireGuard integration and advanced port-forwarding for debugging.

---

## 1. Distro Installation

Arch Linux must be installed as a WSL distribution named `vpn1`. Choose **one** of the two methods below — automated or manual

### Method A: Automated Installation
Install Arch Linux directly from the WSL catalog.

```powershell
wsl --install archlinux --name vpn1
```

### Method B: Manual Installation
Download the latest `.wsl` image from [official mirror](https://wiki.archlinux.org/title/Install_Arch_Linux_on_WSL#Manual_installation), then install it from the file:

```powershell
# Syntax: wsl --install --from-file <PathToWslFile> --name vpn1
wsl --install --from-file archlinux-2026.xxxxx.wsl --name vpn1
```

> Both methods produce the same result a distro named `vpn1`.

### Launching the Distro and update root password
```bash
wsl -d vpn1
passwd
```

### Create the Default User
After first launch, create a regular user inside the distro. **Use a distinct UID for each WSL distribution** — see the warning below.

```bash
# Run inside the vpn1 distro (as root)
useradd -m -u 1010 -G wheel -s /bin/bash <username>
passwd <username>
```

> **Warning — distinct UIDs are required for systemd**
> WSL does not support enabling systemd in multiple running distributions when their respective default users share the same UID. It is recommended to use distinct UIDs for the default user of each WSL distribution/image (e.g. `1010` for `vpn1`, `1011` for the next one, and so on).

Set this new user as the default user for the `vpn1` distribution, then exit and re-launch so the change takes effect:

Install `sudo` and `nano`, then grant `wheel` group members sudo access:

```bash
# Still running as root inside the vpn1 distro
pacman -Sy sudo nano

# Open the sudoers file with nano
EDITOR=nano visudo
```

In `visudo`, find and uncomment the following line (remove the leading `#`):

```sudoers
# %wheel ALL=(ALL:ALL) ALL
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano). Then exit the distro:

```bash
# Exit the distro
exit
```

```powershell
# Run in PowerShell (Windows side)
wsl --manage vpn1 --set-default-user <username>

# Re-launch — you should now be logged in as <username> instead of root
wsl -d vpn1
```

### Maintaining Antigravity (Windows Side)
To keep the Windows-side IDE host up to date, use `winget`:

```powershell
# Check current version
winget list Google.Antigravity

# Upgrade to latest
winget upgrade Google.Antigravity
```

---

## 2. Initial Environment Setup

Execute these commands inside your `vpn1` instance to initialize and update the system:

```bash
# 1. Update system and keys
sudo pacman -Syu

# 2. Install essential system, network, and development tools
sudo pacman -S --needed \
    nano wireguard-tools openresolv \
    tar gzip unzip xz which \
    curl wget jq uv nodejs npm python
```

---

## 3. WireGuard Configuration

Configure your VPN interface to route IDE traffic securely.

### Create the Config File
```bash
sudo nano /etc/wireguard/wg0.conf
```

### Configuration Template (`/etc/wireguard/wg0.conf`)
```ini
[Interface]
Address = 10.1.0.4/32
PrivateKey = <YOUR_PRIVATE_KEY>

[Peer]
Endpoint = <SERVER_IP>:<PORT>
PublicKey = <SERVER_PUBLIC_KEY>
PersistentKeepalive = 25
AllowedIPs = 0.0.0.0/0
```

### Management Commands
```bash
sudo wg-quick up wg0    # Start VPN
sudo wg show wg0        # Check status
sudo wg-quick down wg0  # Stop VPN
```

---

## 4. Automation Scripts (`~/bin`)

Create a `~/bin` directory for custom scripts and ensure they are executable (`chmod +x ~/bin/*`).

### A. The Launch Script (`~/bin/ag`)
Launches the Windows Antigravity executable from within WSL.

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Auto-detect Windows Username
WIN_USER=$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')

if [ -z "$WIN_USER" ]; then
    echo "Error: Could not automatically detect Windows username."
    exit 1
fi

DISTRO="${WSL_DISTRO_NAME:-vpn1}"
AG_EXE="/mnt/c/Users/$WIN_USER/AppData/Local/Programs/Antigravity/bin/antigravity"

# 2. Check if the executable exists
if [ ! -x "$AG_EXE" ]; then
  echo "Antigravity not found at: $AG_EXE"
  echo "Detected Windows User: $WIN_USER"
  exit 1
fi

# 3. Execute with absolute pathing
TARGET="${1:-.}"
ABS_PATH="$(readlink -f "$TARGET")"
"$AG_EXE" --remote "wsl+$DISTRO" "$ABS_PATH"
```

### B. The VPN & Forwarding Script (`~/bin/wg`)
Manages WireGuard and creates an `iptables` tunnel to bridge WSL `9222` to Windows `9223`.

```bash
#!/bin/bash

# 1. Bring up WireGuard
sudo /usr/bin/wg-quick up wg0 2>/dev/null || echo "WireGuard wg0 is already up."

# 2. Identify Windows Host IP
WINDOWS_IP=$(ip route show dev eth0 | grep default | awk '{print $3}' | head -n 1)

if [ -z "$WINDOWS_IP" ]; then
    echo "Error: Could not find Windows Host IP."
    exit 1
fi

# 3. Enable local routing and apply NAT rules
sudo /usr/sbin/sysctl -w net.ipv4.conf.all.route_localnet=1 > /dev/null

# Cleanup existing rules to prevent duplicates
while sudo /usr/sbin/iptables -t nat -D OUTPUT -d 127.0.0.1 -p tcp --dport 9222 -j DNAT --to-destination "$WINDOWS_IP:9223" 2>/dev/null; do :; done
while sudo /usr/sbin/iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE 2>/dev/null; do :; done

# Apply Forwarding (WSL 127.0.0.1:9222 -> Windows:9223)
sudo /usr/sbin/iptables -t nat -A OUTPUT -d 127.0.0.1 -p tcp --dport 9222 -j DNAT --to-destination "$WINDOWS_IP:9223"
sudo /usr/sbin/iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

echo "----------------------------------------------------------"
echo "Target Windows IP: $WINDOWS_IP"
echo "Forwarding: WSL 127.0.0.1:9222 -> Windows:9223"
echo "Status: SUCCESS"
echo "----------------------------------------------------------"
```

---

## 5. Windows Host Configuration (Admin)

Run these in **PowerShell (Admin)** to complete the networking bridge.

### A. Firewall Rule
```powershell
New-NetFirewallRule -DisplayName "WSL to Windows Chrome Debug" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 9223 -RemoteAddress 172.16.0.0/12,192.168.0.0/16
```

### B. Port Proxy
Maps the incoming WSL traffic from port `9223` to your local debugger on `9222`.

```powershell
# Map Windows:9223 -> Windows:9222 (localhost)
netsh interface portproxy add v4tov4 listenport=9223 listenaddress=0.0.0.0 connectport=9222 connectaddress=127.0.0.1

# Verify
netsh interface portproxy show all
```
