# 🌙 MoonNodes VPS Manager Bot

A full-featured Discord bot for managing Incus/LXC VPS containers — with KVM/QEMU support, MoonCoin economy, AF2F admin 2-factor verification, animated responses, and multi-node infrastructure.

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

## ✨ What's New (v3.0.0)

| Area | Change |
|---|---|
| 🖥️ **KVM/QEMU** | Full VM support via `--vm` flag — pick LXC or KVM on `/create` |
| 🌙 **Debian** | Debian 11, 12, 13 fully supported (containers + VMs) |
| 🔐 **AF2F** | Admin 2-Factor verification for sensitive commands |
| 🎬 **Animations** | Named animation sets in `animation/` — deploy, startup, coins, more |
| ⏳ **Expire** | `/mod role::trident: action:Expire` — set/edit/remove VPS expiry |
| 🔒 **Command Lock** | All commands locked until owner completes first-boot setup |
| 🚀 **First Boot** | 10-option setup menu sent to owner on first start |
| 🌙 **MoonCoin** | `/set setting:MoonCoin` — add/remove/view rules, grant/deduct coins |
| 🔧 **Network Fix** | Post-start eth0 verification + automatic NIC re-attach if missing |
| 📝 **DM Commands** | Renamed from "Can DM Commands" → "DM Commands" |
| 🗑️ `/account` | Admin actions removed — moved to `/set` (coins) and `/mod` (VPS) |

---

## 1. Requirements

**Server (host machine):**
- Ubuntu 22.04 / 24.04 **or** Debian 12 / 13 (recommended)
- 64-bit CPU with virtualisation support (Intel VT-x or AMD-V) — required for KVM mode
- Minimum 2 GB RAM, 20 GB disk (more depending on how many VPS you plan to host)
- Root or sudo access

**Software:**
- Python 3.11 or newer
- Incus (recommended) **or** LXD
- `pip` and `git`

**Discord:**
- A Discord bot application with a valid token
- Bot invited to your server with `bot` + `applications.commands` scopes and Administrator permission

---

## 2. Install System Dependencies

Run all of these as root or with `sudo`:

```bash
# Update system
apt update && apt upgrade -y

# Install Python 3.11+ and tools
apt install -y python3 python3-pip python3-venv git curl wget unzip

# Verify Python version (must be 3.11+)
python3 --version

# Install network tools used by the bot
apt install -y iproute2 iputils-ping bridge-utils
```

**For KVM/QEMU VM support**, also install:

```bash
apt install -y qemu-kvm libvirt-daemon-system cpu-checker
kvm-ok   # should print "KVM acceleration can be used"
ls /dev/kvm   # should exist
```

---

## 3. Install the Bot

```bash
# Clone or unzip the bot into a folder
unzip dev.zip -d moonnodes
cd moonnodes/dev_new

# Create a virtual environment
python3 -m venv venv
source venv/bin/activate

# Install Python dependencies
pip install -r requirements.txt
```

**requirements.txt includes:**
- `discord.py >= 2.3.0`
- `python-dotenv >= 1.0.0`
- `aiohttp >= 3.9.0`

---

## 4. Install Incus (Recommended)

Incus is the recommended backend. It is the community fork of LXD and actively maintained.

```bash
# Add Zabbly repository (provides latest Incus builds)
curl -fsSL https://pkgs.zabbly.com/key.asc | gpg --dearmor -o /etc/apt/keyrings/zabbly.gpg
sh -c 'cat > /etc/apt/sources.list.d/zabbly-incus-stable.sources << EOF
Enabled: yes
Types: deb
URIs: https://pkgs.zabbly.com/incus/stable
Suites: $(. /etc/os-release && echo ${VERSION_CODENAME})
Components: main
Archive-Key: /etc/apt/keyrings/zabbly.gpg
EOF'

apt update
apt install -y incus

# Initialise Incus (accept defaults or customise)
incus admin init --minimal

# Verify
incus version
incus launch images:ubuntu/22.04 test-container
incus list
incus delete test-container --force
```

**Add your user to the incus group** (so the bot process can call `incus` without root):

```bash
usermod -aG incus $USER
newgrp incus
```

**Verify the binary path** — the bot auto-detects `/usr/bin/incus`:

```bash
which incus   # should print /usr/bin/incus
```

---

## 5. Install LXD (Alternative)

Use LXD only if you can't use Incus. LXD is maintained by Canonical and available via Snap.

```bash
snap install lxd
lxd init --minimal

# Add your user
usermod -aG lxd $USER
newgrp lxd

# Verify
lxc version
```

> ⚠️ KVM/QEMU VM mode requires Incus. LXD VM support is limited and not tested.

---

## 6. Multi-Node Setup

Multi-node lets you deploy VPS on remote machines from a single bot. Each remote node runs Incus and exposes its API.

**On each remote node:**

```bash
# Install Incus (same steps as Section 4)
# Then expose the Incus API
incus config set core.https_address :8443

# Create a trust certificate for the bot host
incus config trust add bot-main-node
# Copy the printed token — you'll need it when adding the node in Discord
```

**On the main bot host:**

```bash
# Add the remote node (replace with actual IP and token)
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

Then `/set` → channels and the bot will balance deployments across nodes.

---

## 7. Configure .env

Create a `.env` file in the bot root (same folder as `bot.py`):

```bash
cp .env.example .env   # if example exists
# OR create it manually:
nano .env
```

### Required

```env
DISCORD_TOKEN=your_discord_bot_token_here
BOT_OWNER_ID=your_discord_user_id_here
GUILD_ID=your_discord_server_id_here
```

### Role IDs
Set these to your Discord server's role IDs. Right-click a role → Copy ID (Developer Mode must be on).

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
VPS_PREFIX=vps               # Prefix for container names e.g. vps-john-1
DEFAULT_STORAGE_POOL=        # Leave blank to auto-detect, or set e.g. "default"
ENFORCE_RESOURCE_LIMITS=True # Cap RAM/CPU at MAX_RAM_GB / MAX_CPU_CORES
MAX_RAM_GB=16
MAX_CPU_CORES=8
HOST_PUBLIC_IP=              # Your server's public IP (shown in SSH info)
```

### Thresholds & Alerts

```env
CPU_THRESHOLD=90   # Alert when container CPU % exceeds this
RAM_THRESHOLD=90   # Alert when container RAM % exceeds this
```

### DM Commands

```env
DM_COMMANDS=False   # Set True to allow users to run commands via DM
DM_PREFIX=!         # Prefix for DM commands (e.g. !list)
```

### Appearance

```env
BOT_COLOR=5865F2   # Embed accent colour (hex, no #)
```

### Cloudflare (optional — for auto subdomain)

```env
ASM=False             # Set True to enable auto subdomain assignment
CF_API_TOKEN=         # Cloudflare API token
CF_ZONE_ID=           # Cloudflare Zone ID
CF_DOMAIN=moonlink.qzz.io   # Domain to create subdomains under
```

### Extensions (can also be toggled via `/system action:Toggle`)

```env
Multi-Node=False
MARKETPLACE=True
ACCOUNT=True
VPS_RENEWAL=False
VPS_MONITORING=False
ISTS=False              # Ticket System
AES=False               # Auto Expire & Suspend
RR=False                # Renewal Reminders
A2FA=False              # Bot 2FA
MSTC=False              # Multi-Server Tickets
SOSB=False              # Scheduled Backups
UBL_ENABLED=False       # Backup Limits
MULTILANG=False
DM_COMMANDS=False
BLOCK_TERMINAL=False
BACKUP_LIMIT=1
MAIN_CHAT_ID=           # Channel ID for main bot activity channel
```

---

## 8. Run the Bot

```bash
# Make sure your venv is active
source venv/bin/activate

# Run
python3 bot.py
```

**Run as a background service (recommended for production):**

```bash
# Create a systemd service
cat > /etc/systemd/system/moonnodes.service << EOF
[Unit]
Description=MoonNodes VPS Manager Bot
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/path/to/moonnodes/dev_new
ExecStart=/path/to/moonnodes/dev_new/venv/bin/python3 bot.py
Restart=always
RestartSec=10
EnvironmentFile=/path/to/moonnodes/dev_new/.env

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable moonnodes
systemctl start moonnodes

# Check status
systemctl status moonnodes
journalctl -u moonnodes -f   # live logs
```

---

## 9. First Boot

On first run, the bot sends the **owner** a DM with a setup menu. **All commands are locked for everyone else until the owner clicks a button.**

| Button | What it does |
|---|---|
| 📤 **Restore system.db** | Upload your existing `system.db` — all users, VPS, coins restored |
| ✅ **Start Fresh** | Clean database, configure channels from scratch |
| ⚙️ **Quick Setup** | Fresh start with a channel config checklist |
| 🔗 **Restore from URL** | Paste a direct download URL to `system.db` |
| 🌐 **Multi-Node Setup** | Fresh start, then immediately prompts to add remote nodes |
| 📂 **Import vps_data.json** | Migrate from a legacy JSON backup |
| 🎭 **Demo Mode** | Explore the bot safely — no real VPS created until you run `/create` |
| ⚡ **Minimal Mode** | Core VPS only, all extensions off |
| ⏭️ **Skip** | Unlock commands immediately, configure later |
| 🔄 **Copy from Another Bot** | Step-by-step instructions to clone another running instance |

> The 10-minute window resets each time the bot restarts until you complete setup.
> To re-run first boot: delete the `.first_boot_done` file in the bot folder and restart.

---

## 10. First Steps in Discord

After first boot is complete:

**1. Set up channels**

```
/set setting:Log Channel value:<channel_id>
/set setting:Main Chat Channel value:<channel_id>
/set setting:Payment Channel value:<channel_id>
/set setting:AF2F Channel value:<channel_id>
```

**2. Set up roles** (if not already in `.env`)

```
/role action:Add role:Admin discord_role:@YourAdminRole
/role action:Add role:Mod discord_role:@YourModRole
```

**3. Enable extensions you want**

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
| `/create` | Deploy a VPS for a user — choose node, OS, and LXC or KVM |
| `/delete` | Permanently delete a VPS |
| `/af2f code:` | Submit AF2F 2-factor code to authorise a pending action |
| `/node action:` | Add, edit, remove, list, or ping remote Incus nodes |
| `/system action:` | Toggle extensions, view bot status, manage settings |
| `/admin action:` | Low-level admin tools — DB queries, cache flush, restart |

### 🛡️ Admin

| Command | Description |
|---|---|
| `/set setting:` | Configure all bot settings — channels, thresholds, MoonCoin rules, AF2F |
| `/manage user_id:` | Admin view of any user's VPS panel |
| `/list action:All VPS` | All VPS across all users |
| `/giveaway action:` | Create, manage, or end VPS giveaways |

### 🔱 Hard Mod (Trident) — AF2F Required

| Command | Description |
|---|---|
| `/mod role::trident: action:Suspend VPS` | Suspend a user's VPS 🔐 |
| `/mod role::trident: action:Unsuspend VPS` | Unsuspend a user's VPS 🔐 |
| `/mod role::trident: action:Expire` | Set, edit, remove, or view VPS expiry 🔐 |
| `/mod role::trident: action:Blocklist` | Add/remove users from blocklist |
| `/mod role::trident: action:Mute` | Mute/unmute users |
| `/mod role::trident: action:Broadcast` | Send announcement to all VPS users |
| `/mod role::trident: action:Lock Channel` | Lock a Discord channel |
| `/mod role::trident: action:Purge` | Bulk delete messages |

### 🛡️ Mod

| Command | Description |
|---|---|
| `/mod role:Mod action:User Info` | View a user's full profile |
| `/mod role:Mod action:VPS Info` | View details for a specific VPS |
| `/mod role:Mod action:List VPS` | List all VPS for a user |
| `/mod role:Mod action:Search VPS` | Search VPS by container name |
| `/mod role:Mod action:Server Stats` | Live host resource stats |
| `/mod role:Mod action:DM User` | Send a DM to a user |
| `/mod role:Mod action:Warn User` | Issue a warning |
| `/mod role:Mod action:Warn Log` | View warning history |

### 👤 User

| Command | Description |
|---|---|
| `/list` | List your own VPS |
| `/manage` | Full VPS management panel — SSH info, start/stop, backup, upgrade |
| `/account` | MoonCoin balance, transfer coins, VPS share/backup |
| `/plans` | View available VPS plans |
| `/claim` | Claim a free or premium VPS (if enabled) |
| `/status` | Bot and host status |
| `/support` | Open a support ticket |
| `/marketplace` | Buy/sell VPS on the marketplace |
| `/help` | Full command guide |

### 🔐 AF2F Protected Commands

The following always require AF2F verification for non-owner staff:

`/create` · `/delete` · `/set` · `/system` · `/node` · `/role` · `/mod action:Expire` · `/mod action:Suspend VPS` · `/mod action:Unsuspend VPS`

> The **bot owner** (`BOT_OWNER_ID`) is always exempt from AF2F.

---

## 12. Extensions Reference

Toggle any extension on/off with:

```
/system action:Toggle extension:<Extension Name>
```

Or set in `.env` and restart.

| Extension | Env Key | Description |
|---|---|---|
| Multi-Node Support | `Multi-Node` | Deploy VPS across multiple remote Incus nodes |
| Scheduled Backups | `SOSB` | Auto-snapshot VPS on a schedule |
| Ticket System | `ISTS` | In-server support ticket system |
| Auto Expire & Suspend | `AES` | Automatically suspend/expire VPS past due date |
| Renewal Reminders | `RR` | DM users before their VPS expires |
| Bot 2FA | `A2FA` | General 2FA for bot actions (separate from AF2F) |
| Multi-Server Tickets | `MSTC` | Route tickets across multiple Discord servers |
| Backup Limits | `UBL_ENABLED` | Cap how many backups a user can store (`BACKUP_LIMIT`) |
| Marketplace | `MARKETPLACE` | Enable VPS marketplace for peer-to-peer transfers |
| Account & MoonCoin | `ACCOUNT` | Enable MoonCoin economy and `/account` command |
| VPS Renewal | `VPS_RENEWAL` | Let users renew their VPS via payment |
| VPS Monitoring Alerts | `VPS_MONITORING` | Alert staff when CPU/RAM exceeds threshold |
| Multi-language | `MULTILANG` | Enable multi-language bot responses |
| DM Commands | `DM_COMMANDS` | Allow users to run commands via DM |
| Block Terminal Commands | `BLOCK_TERMINAL` | Prevent specific terminal commands from running inside VPS |

---

## 13. Transfer Data

### Move to a new server

```bash
# On old server — back up the database
cp system.db system.db.backup
sqlite3 system.db .dump > backup.sql

# Transfer to new server
scp system.db user@new-server:/path/to/moonnodes/dev_new/

# On new server — start the bot
# On first boot, choose 📤 Restore system.db and upload the file
```

### Migrate from legacy vps_data.json

If you are coming from an older version of the bot that stored data in JSON:

```bash
# Put vps_data.json in the bot root folder
cp /old/path/vps_data.json /path/to/moonnodes/dev_new/

# On first boot, choose 📂 Import vps_data.json
# The bot will migrate all records to system.db automatically
```

### Reset first-boot (re-run setup)

```bash
rm /path/to/moonnodes/dev_new/.first_boot_done
systemctl restart moonnodes
```

The owner will receive the setup DM again on next start.

---

## 14. Update the Bot

```bash
# Stop the service
systemctl stop moonnodes

# Back up data
cp system.db system.db.pre-update.bak

# Replace bot files (unzip new version over existing folder)
unzip -o dev_new.zip -d /path/to/moonnodes/

# Re-install dependencies in case requirements changed
source venv/bin/activate
pip install -r requirements.txt

# Restart
systemctl start moonnodes
systemctl status moonnodes
```

> The database schema auto-migrates on startup — new tables are created if they don't exist. Your existing data is never deleted on update.

---

## 15. Troubleshooting

### Bot doesn't start

```bash
# Check logs
journalctl -u moonnodes -n 50

# Common causes:
# - DISCORD_TOKEN missing or invalid in .env
# - BOT_OWNER_ID not set
# - Python version < 3.11 (check: python3 --version)
# - Missing dependency (re-run: pip install -r requirements.txt)
```

### Commands not showing in Discord

```bash
# Commands are registered as slash commands on startup.
# If they don't appear, wait 1-2 minutes then try:
# 1. Restart the bot
# 2. Kick and re-invite the bot with applications.commands scope
# 3. Check GUILD_ID is set correctly in .env
```

### VPS gets "Network is unreachable"

The bot now auto-recovers from this. If it still happens, run manually on the host:

```bash
# Check if bridge exists
ip link show incusbr0

# If missing, create it
incus network create incusbr0

# Re-attach NIC to the container
incus config device remove <container_name> eth0
incus config device add <container_name> eth0 nic nictype=bridged parent=incusbr0
incus restart <container_name>

# Verify
incus exec <container_name> -- ping -c 3 8.8.8.8
```

### KVM VM won't start

```bash
# Check KVM is available
ls /dev/kvm          # must exist
kvm-ok               # should say "KVM acceleration can be used"

# Check Incus supports VMs
incus info | grep -A5 "driver:"

# If /dev/kvm is missing, enable virtualisation in your VPS host's control panel
# (Hetzner: enable "KVM virtualisation" in server settings)
# (Contabo: enable nested virtualisation in control panel)
```

### AF2F code not arriving

```bash
# 1. Make sure the AF2F channel is set:
#    /set setting:AF2F Channel value:<channel_id>

# 2. Make sure the bot has permission to send messages in that channel

# 3. The code expires in 5 minutes — re-run the original command to get a new one

# 4. Bot owner is exempt — if you are the owner, you don't need AF2F
```

### First-boot DM not received

```bash
# Make sure you have DMs open from server members
# Check BOT_OWNER_ID matches your actual Discord user ID exactly
# Check bot logs for: "First-boot DM failed"

# Force re-send by deleting the flag file and restarting:
rm /path/to/moonnodes/dev_new/.first_boot_done
systemctl restart moonnodes
```

### Incus not found / wrong binary

The bot auto-detects Incus at `/usr/bin/incus` and `/usr/local/bin/incus`. If your binary is elsewhere:

```bash
which incus   # find the path
# Then symlink it:
ln -s $(which incus) /usr/bin/incus
```

### Check runtime detection

```bash
# In Discord (owner only):
/system action:Status
# Shows: Runtime (Incus/LXD), storage pool, node count, extension states
```

---

*MoonNodes VPS Manager — Built for Discord VPS hosting communities.*
