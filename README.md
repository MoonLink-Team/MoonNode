<div align="center">

# 🌙 MoonNodes VPS Manager

**A full-featured Discord bot for managing Incus/LXC VPS containers**

KVM/QEMU support · MoonCoin economy · AF2F admin 2-factor · animated responses · multi-node infrastructure

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)](https://python.org)
[![discord.py](https://img.shields.io/badge/discord.py-2.3+-5865F2?style=flat&logo=discord&logoColor=white)](https://github.com/Rapptz/discord.py)
[![Backend](https://img.shields.io/badge/Backend-Incus%20%2F%20LXD-orange?style=flat)](https://linuxcontainers.org/incus/)

```bash
git clone https://github.com/MoonLink-Team/MoonNode
```

</div>

---

## 📋 Table of Contents

1. [Requirements](#1-requirements)
2. [Install System Dependencies](#2-install-system-dependencies)
3. [Install the Bot](#3-install-the-bot)
4. [Install Incus (Recommended)](#4-install-incus-recommended)
5. [Install LXD (Alternative)](#5-install-lxd-alternative)
6. [Multi-Node Setup](#6-multi-node-setup)
7. [Configure .env](#7-configure-env)
8. [Run the Bot](#8-run-the-bot)
9. [First Boot](#9-first-boot)
10. [First Steps in Discord](#10-first-steps-in-discord)
11. [Commands Reference](#11-commands-reference)
12. [Extensions Reference](#12-extensions-reference)
13. [Transfer Data](#13-transfer-data)
14. [Update the Bot](#14-update-the-bot)
15. [Troubleshooting](#15-troubleshooting)

---

## ✨ What's New — v3.0.0

| Area | Change |
|---|---|
| 🖥️ **KVM/QEMU** | Full VM support via `--vm` flag — choose LXC or KVM on `/create` |
| 🌙 **Debian** | Debian 11, 12, 13 fully supported (containers + VMs) |
| 🔐 **AF2F** | Admin 2-Factor verification for all sensitive commands |
| 🎬 **Animations** | Named animation sets in `animation/` — deploy, startup, coins, and more |
| ⏳ **Expire** | `/mod role::trident: action:Expire` — set / edit / remove VPS expiry |
| 🔒 **Command Lock** | All commands locked until owner completes first-boot setup |
| 🚀 **First Boot** | 10-option setup menu sent to owner on first start |
| 🌙 **MoonCoin** | `/set setting:MoonCoin` — add / remove / view rules, grant / deduct coins |
| 🔧 **Network Fix** | Post-start eth0 verification + automatic NIC re-attach if missing |
| 📝 **DM Commands** | Renamed from "Can DM Commands" → "DM Commands" |
| 🗑️ `/account` | Admin actions removed — moved to `/set` (coins) and `/mod` (VPS) |

---

## 1. Requirements

### Host Machine
- Ubuntu 22.04 / 24.04 **or** Debian 12 / 13 (recommended)
- 64-bit CPU with virtualisation support (Intel VT-x or AMD-V) — required for KVM mode
- Minimum 2 GB RAM, 20 GB disk (scale up per planned VPS count)
- Root or sudo access

### Software
- Python 3.11 or newer
- Incus (recommended) **or** LXD
- `pip` and `git`

### Discord
- A Discord bot application with a valid token
- Bot invited with `bot` + `applications.commands` scopes and **Administrator** permission

---

## 2. Install System Dependencies

Run as root or with `sudo`:

```bash
# Update system
apt update && apt upgrade -y

# Python 3.11+ and tools
apt install -y python3 python3-pip python3-venv git curl wget unzip

# Verify version (must be 3.11+)
python3 --version

# Network tools used by the bot
apt install -y iproute2 iputils-ping bridge-utils
```

**For KVM/QEMU VM support**, also install:

```bash
apt install -y qemu-kvm libvirt-daemon-system cpu-checker
kvm-ok            # should print "KVM acceleration can be used"
ls /dev/kvm       # should exist
```

---

## 3. Install the Bot

```bash
# Clone the repository
git clone https://github.com/MoonLink-Team/MoonNode
cd MoonNode

# Make the dev.zip to unzip
apt install zip
unzip

# Create a virtual environment
python3 -m venv venv
source venv/bin/activate

# Install Python dependencies
pip install -r requirements.txt
```

**`requirements.txt` includes:**
- `discord.py >= 2.3.0`
- `python-dotenv >= 1.0.0`
- `aiohttp >= 3.9.0`

---

## 4. Install Incus (Recommended)

Incus is the recommended backend — the community fork of LXD, actively maintained.

```bash
# bash
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sh
incus admin init --auto

# Profile
incus profile set default security.privileged true
incus profile set default security.nesting true
incus profile set default security.syscalls.intercept.mknod true

# Storage (pick one)
incus storage create default dir
# incus storage create default zfs   ← recommended for production

# Network
incus network create incusbr0
incus profile device add default eth0 nic nictype=bridged parent=incusbr0

# Test
incus launch images:ubuntu/22.04 test && incus list && incus delete test --force
```

**Add your user to the incus group:**

```bash
usermod -aG incus-admin $USER
newgrp incus-admin
```

**Verify binary path** (bot auto-detects `/usr/bin/incus`):

```bash
which incus   # should print /usr/bin/incus
```

---

## 5. Install LXD (Alternative)

Use LXD only if Incus is not available. LXD is available via Snap.

```bash
snap install lxd
lxd init --minimal

# Add your user
usermod -aG lxd $USER
newgrp lxd

# Verify
lxc version
```

> ⚠️ KVM/QEMU VM mode requires Incus. LXD VM support is limited and untested.

---

## 6. Multi-Node Setup

Multi-node lets you deploy VPS on remote machines from a single bot instance.

**On each remote node:**

```bash
# Install Incus (same steps as Section 4)
# Expose the Incus API
incus config set core.https_address :8443

# Create a trust token for the main bot host
incus config trust add bot-main-node
# Copy the printed token — needed when adding the node in Discord
```

**On the main bot host:**

```bash
# Add the remote node
incus remote add node2 https://<remote-ip>:8443 --token <token>
incus remote list   # should show node2
```

**In Discord**, after the bot is running:

```
/node action:Add name:node2 runtime:incus api_key:<token>
```

Enable the extension in `.env`:

```env
Multi-Node=True
```

---

## 7. Configure .env

Create a `.env` file in the bot root (same folder as `bot.py`):

```bash
cp .env.example .env
nano .env
```

### Required

```env
DISCORD_TOKEN=your_discord_bot_token_here
BOT_OWNER_ID=your_discord_user_id_here
GUILD_ID=your_discord_server_id_here
```

### Role IDs

Right-click a role → **Copy ID** (requires Developer Mode).

```env
OWNER_ROLE_ID=
HARD_ADMIN_ROLE_ID=
ADMIN_ROLE_ID=
MANAGER_ROLE_ID=
HARD_MOD_ROLE_ID=
MOD_ROLE_ID=
HARD_STAFF_ROLE_ID=
STAFF_ROLE_ID=
TRIAL_STAFF_ROLE_ID=
TRIAL_MOD_ROLE_ID=
PARTNER_ROLE_ID=
VIP_ROLE_ID=
VPS_USER_ROLE_ID=
```

### VPS Settings

```env
VPS_PREFIX=vps
DEFAULT_STORAGE_POOL=
ENFORCE_RESOURCE_LIMITS=True
MAX_RAM_GB=16
MAX_CPU_CORES=8
HOST_PUBLIC_IP=
```

### Thresholds & Alerts

```env
CPU_THRESHOLD=90
RAM_THRESHOLD=90
```

### Appearance

```env
BOT_COLOR=5865F2
```

### Cloudflare — Auto Subdomain (optional)

```env
ASM=False
CF_API_TOKEN=
CF_ZONE_ID=
CF_DOMAIN=moonlink.qzz.io
```

### Extensions

```env
Multi-Node=False
MARKETPLACE=True
ACCOUNT=True
VPS_RENEWAL=False
VPS_MONITORING=False
ISTS=False
AES=False
RR=False
A2FA=False
MSTC=False
SOSB=False
UBL_ENABLED=False
MULTILANG=False
DM_COMMANDS=False
BLOCK_TERMINAL=False
BACKUP_LIMIT=1
MAIN_CHAT_ID=
```

---

## 8. Run the Bot

```bash
source venv/bin/activate
python3 bot.py
```

**As a systemd service (recommended):**

```bash
cat > /etc/systemd/system/moonnodes.service << EOF
[Unit]
Description=MoonNodes VPS Manager Bot
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/MoonNode
ExecStart=/root/MoonNode/venv/bin/python3 bot.py
Restart=always
RestartSec=10
EnvironmentFile=/root/MoonNode/.env

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable moonnodes
systemctl start moonnodes
systemctl status moonnodes
journalctl -u moonnodes -f
```

---

## 9. First Boot

On first run, the bot DMs the **owner** a setup menu. **All commands are locked until setup is complete.**

| Button | What it does |
|---|---|
| 📤 **Restore system.db** | Upload existing `system.db` — all users, VPS, coins restored |
| ✅ **Start Fresh** | Clean database, configure channels from scratch |
| ⚙️ **Quick Setup** | Fresh start with a channel config checklist |
| 🔗 **Restore from URL** | Paste a direct download URL to `system.db` |
| 🌐 **Multi-Node Setup** | Fresh start, then prompts to add remote nodes |
| 📂 **Import vps_data.json** | Migrate from a legacy JSON backup |
| 🎭 **Demo Mode** | Explore safely — no real VPS created until `/create` |
| ⚡ **Minimal Mode** | Core VPS only, all extensions off |
| ⏭️ **Skip** | Unlock commands immediately, configure later |
| 🔄 **Copy from Another Bot** | Step-by-step to clone another running instance |

> To re-run first boot: delete `.first_boot_done` in the bot folder and restart.

---

## 10. First Steps in Discord

**1. Set up channels**

```
/set setting:Log Channel value:<channel_id>
/set setting:Main Chat Channel value:<channel_id>
/set setting:Payment Channel value:<channel_id>
/set setting:AF2F Channel value:<channel_id>
```

**2. Set up roles**

```
/role action:Add role:Admin discord_role:@YourAdminRole
/role action:Add role:Mod discord_role:@YourModRole
```

**3. Enable extensions**

```
/system action:Toggle extension:Ticket System
/system action:Toggle extension:VPS Monitoring Alerts
/system action:Toggle extension:Account & MoonCoin
```

**4. Deploy a test VPS**

```
/create user:@yourself plan_name:basic vm_mode:LXC Container
```

**5. Check status**

```
/system action:Status
/list action:All VPS
```

---

## 11. Commands Reference

### 👑 Owner

| Command | Description |
|---|---|
| `/create` | Deploy a VPS for a user — choose node, OS, LXC or KVM |
| `/delete` | Permanently delete a VPS |
| `/af2f code:` | Submit AF2F 2-factor code |
| `/node action:` | Add, edit, remove, list, or ping remote Incus nodes |
| `/system action:` | Toggle extensions, view status, manage settings |
| `/admin action:` | Low-level admin tools — DB queries, cache flush, restart |

### 🛡️ Admin

| Command | Description |
|---|---|
| `/set setting:` | Configure all bot settings |
| `/manage user_id:` | Admin view of any user's VPS panel |
| `/list action:All VPS` | All VPS across all users |
| `/giveaway action:` | Create, manage, or end VPS giveaways |

### 🔱 Hard Mod (Trident) — AF2F Required

| Command | Description |
|---|---|
| `/mod role::trident: action:Suspend VPS` | Suspend a VPS 🔐 |
| `/mod role::trident: action:Unsuspend VPS` | Unsuspend a VPS 🔐 |
| `/mod role::trident: action:Expire` | Set / edit / remove VPS expiry 🔐 |
| `/mod role::trident: action:Blocklist` | Add / remove users from blocklist |
| `/mod role::trident: action:Mute` | Mute / unmute users |
| `/mod role::trident: action:Broadcast` | Announce to all VPS users |
| `/mod role::trident: action:Lock Channel` | Lock a Discord channel |
| `/mod role::trident: action:Purge` | Bulk delete messages |

### 🛡️ Mod

| Command | Description |
|---|---|
| `/mod role:Mod action:User Info` | View a user's full profile |
| `/mod role:Mod action:VPS Info` | View details for a specific VPS |
| `/mod role:Mod action:List VPS` | List all VPS for a user |
| `/mod role:Mod action:Search VPS` | Search by container name |
| `/mod role:Mod action:Server Stats` | Live host resource stats |
| `/mod role:Mod action:DM User` | Send a DM to a user |
| `/mod role:Mod action:Warn User` | Issue a warning |
| `/mod role:Mod action:Warn Log` | View warning history |

### 👤 User

| Command | Description |
|---|---|
| `/list` | List your own VPS |
| `/manage` | Full VPS management panel |
| `/account` | MoonCoin balance, transfer coins, share/backup |
| `/plans` | View available VPS plans |
| `/claim` | Claim a free or premium VPS |
| `/status` | Bot and host status |
| `/support` | Open a support ticket |
| `/marketplace` | Buy / sell VPS |
| `/help` | Full command guide |

### 🔐 AF2F Protected Commands

`/create` · `/delete` · `/set` · `/system` · `/node` · `/role` · `/mod action:Expire` · `/mod action:Suspend VPS` · `/mod action:Unsuspend VPS`

> **Bot owner** (`BOT_OWNER_ID`) is always exempt from AF2F.

---

## 12. Extensions Reference

| Extension | Env Key | Description |
|---|---|---|
| Multi-Node Support | `Multi-Node` | Deploy VPS across multiple remote Incus nodes |
| Scheduled Backups | `SOSB` | Auto-snapshot VPS on a schedule |
| Ticket System | `ISTS` | In-server support ticket system |
| Auto Expire & Suspend | `AES` | Automatically suspend / expire past-due VPS |
| Renewal Reminders | `RR` | DM users before their VPS expires |
| Bot 2FA | `A2FA` | General 2FA for bot actions |
| Multi-Server Tickets | `MSTC` | Route tickets across multiple Discord servers |
| Backup Limits | `UBL_ENABLED` | Cap backups per user (`BACKUP_LIMIT`) |
| Marketplace | `MARKETPLACE` | Peer-to-peer VPS transfers |
| Account & MoonCoin | `ACCOUNT` | MoonCoin economy and `/account` command |
| VPS Renewal | `VPS_RENEWAL` | Let users renew via payment |
| VPS Monitoring Alerts | `VPS_MONITORING` | Alert staff when CPU/RAM exceeds threshold |
| Multi-language | `MULTILANG` | Multi-language bot responses |
| DM Commands | `DM_COMMANDS` | Allow commands via DM |
| Block Terminal Commands | `BLOCK_TERMINAL` | Block specific terminal commands inside VPS |

---

## 13. Transfer Data

### Move to a new server

```bash
# Back up the database on old server
cp system.db system.db.backup
scp system.db user@new-server:/root/MoonNode/

# On new server — first boot → 📤 Restore system.db
```

### Migrate from legacy vps_data.json

```bash
cp /old/path/vps_data.json /root/MoonNode/
# On first boot → 📂 Import vps_data.json
```

### Reset first-boot

```bash
rm /root/MoonNode/.first_boot_done
systemctl restart moonnodes
```

---

## 14. Update the Bot

```bash
systemctl stop moonnodes
cp system.db system.db.pre-update.bak

cd /root/MoonNode
git pull origin main

source venv/bin/activate
pip install -r requirements.txt

systemctl start moonnodes
systemctl status moonnodes
```

> Database schema auto-migrates on startup. Existing data is never deleted.

---

## 15. Troubleshooting

### Bot doesn't start

```bash
journalctl -u moonnodes -n 50
# Common: missing DISCORD_TOKEN, BOT_OWNER_ID, Python < 3.11
```

### Commands not showing in Discord

1. Wait 1–2 minutes after start
2. Restart the bot
3. Kick and re-invite with `applications.commands` scope
4. Verify `GUILD_ID` in `.env`

### VPS gets "Network is unreachable"

The bot auto-attempts recovery. If it still fails, run manually:

```bash
# Safe — works even if network/device already exist
incus network show incusbr0 2>/dev/null || incus network create incusbr0

incus config device remove <container_name> eth0 2>/dev/null || true
incus config device add <container_name> eth0 nic nictype=bridged parent=incusbr0

incus restart <container_name>
incus exec <container_name> -- ping -c 3 8.8.8.8
```

### "Error: The device already exists" / "Network already exists"

Bug fixed in v3.0.1. The bot now checks before creating. Update with `git pull` and use the idempotent commands above.

### KVM VM won't start

```bash
ls /dev/kvm     # must exist
kvm-ok          # "KVM acceleration can be used"
incus info | grep -A5 "driver:"
# If /dev/kvm missing → enable nested virtualisation in your host panel
```

### AF2F code not arriving

1. `/set setting:AF2F Channel value:<channel_id>`
2. Confirm bot has send-message permission in that channel
3. Code expires in 5 minutes — re-run original command to get a new one

### First-boot DM not received

```bash
# Ensure DMs open from server members
# Check BOT_OWNER_ID matches your Discord user ID exactly
journalctl -u moonnodes | grep "First-boot"

# Force re-send:
rm /root/MoonNode/.first_boot_done
systemctl restart moonnodes
```

### Incus not found

```bash
which incus
ln -s $(which incus) /usr/bin/incus
```

### Check runtime status

```
/system action:Status
```

---

<div align="center">

*MoonNodes VPS Manager — Built for Discord VPS hosting communities.*

[GitHub](https://github.com/MoonLink-Team/MoonNode) · [Report Issue](https://github.com/MoonLink-Team/MoonNode/issues)

</div>
