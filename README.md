# 🌙 MoonNodes VPS Manager

> Discord bot for managing **LXD / Incus** containers. Slash commands, multi-node clusters, MoonCoin economy, marketplace, staff hierarchy, automated backups, and more. All data in one file: **system.db**.

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

## 1. Requirements

| Item | Version |
|---|---|
| Python | 3.11+ |
| discord.py | 2.3+ |
| Incus **or** LXD | Incus 0.5+ / LXD 5.x |
| OS | Ubuntu 22.04 / 24.04 |
| RAM | 2 GB minimum |
| Disk | 20 GB minimum |
| git | Any |

---

## 2. Install System Dependencies

```bash
# Update system
apt update && apt upgrade -y

# Core packages
apt install -y python3 python3-pip git zip unzip curl screen nano ufw

# Python deps
pip3 install discord.py python-dotenv aiohttp

# Allow SSH (if not already open)
ufw allow 22/tcp
ufw enable
```

---

## 3. Install the Bot

```bash
# Clone from GitHub
git clone https://github.com/MoonLink-Team/MoonNode
cd MoonNode

# OR if you have a zip file:
apt install -y zip
unzip dev.zip

# Install Python requirements
pip3 install -r requirements.txt

# Copy and edit config
cp .env.example .env
nano .env
```

---

## 4. Install Incus (Recommended)

Incus is the recommended runtime — it's the community fork of LXD with active development.

```bash
# Install
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sh
incus admin init --auto

# Add bot user to incus-admin group
usermod -aG incus-admin $USER
newgrp incus-admin

# Configure profile for privileged containers
incus profile set default security.privileged true
incus profile set default security.nesting true
incus profile set default security.syscalls.intercept.mknod true

# Create storage pool (choose one)
incus storage create default dir             # Simple, no extra setup
# incus storage create default zfs          # Recommended for production
# incus storage create default btrfs        # Good alternative

# Create network bridge (VPS internet access)
incus network create incusbr0
incus profile device add default eth0 nic nictype=bridged parent=incusbr0

# Test everything works
incus launch images:ubuntu/22.04 test-vps
incus list
incus exec test-vps -- ping -c 3 8.8.8.8
incus delete test-vps --force
```

> **Important:** The bot deploys containers in two steps — first `incus init`, then separately attaches the root disk and network device. This avoids the "Invalid device type" and "No root device" errors seen with inline `--device` flags.

---

## 5. Install LXD (Alternative)

```bash
# Install
snap install lxd --channel=latest/stable
lxd init --auto

# Add bot user
usermod -aG lxd $USER
newgrp lxd

# Profile
lxc profile set default security.privileged true
lxc profile set default security.nesting true
lxc profile set default security.syscalls.intercept.mknod true

# Storage
lxc storage create default dir

# Network
lxc network create lxdbr0
lxc profile device add default eth0 nic nictype=bridged parent=lxdbr0

# Test
lxc launch ubuntu:22.04 test-vps
lxc list
lxc delete test-vps --force
```

---

## 6. Multi-Node Setup

Multi-node lets you run VPS containers across multiple servers. The bot routes deployment to the right node automatically.

### Incus Multi-Node

**On each REMOTE node:**

```bash
# Install Incus
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sh
incus admin init --auto

# Enable remote API
incus config set core.https_address [::]:8443

# Generate trust token for main server
incus config trust add --name moonbot
# ← COPY the printed token!

# Open firewall
ufw allow 8443/tcp

# Storage + profile
incus storage create default dir
incus profile set default security.privileged true
incus profile set default security.nesting true
incus network create incusbr0
incus profile device add default eth0 nic nictype=bridged parent=incusbr0
```

**On the MAIN server (where bot runs):**

```bash
# Add the remote (paste token when prompted)
incus remote add node2 https://REMOTE_IP:8443

# Verify connection
incus list node2:

# Test deployment
incus launch images:ubuntu/22.04 node2:test-vps
incus delete node2:test-vps --force
```

**Register in Discord:**

```
/node pick_one:Create  →  Fill: Name, API Key (token), Runtime: incus, Section: all/free/premium
/system action:Extension Toggle  extension:Multi-Node Support  ext_state:Enable
```

### LXD Multi-Node

**On each REMOTE node:**

```bash
snap install lxd --channel=latest/stable
lxd init --auto

lxc config set core.https_address [::]:8443
lxc config set core.trust_password "YourPassword"
ufw allow 8443/tcp

lxc storage create default dir
lxc profile set default security.privileged true
lxc profile set default security.nesting true
lxc network create lxdbr0
lxc profile device add default eth0 nic nictype=bridged parent=lxdbr0
```

**On the MAIN server:**

```bash
lxc remote add node2 https://REMOTE_IP:8443 --accept-certificate
# Enter password when prompted
lxc list node2:
```

### Node Sections

When creating a node you set its **Section** — which plan types can deploy on it:

| Section | Who sees it |
|---|---|
| `all` | Everyone (admin + free + premium) |
| `free` | 🌙 MoonCoin, Invite, and Giveaway claims only |
| `premium` | 💎 Premium plan claims only |
| `local` | 🔒 Admin Only — never shown to regular users |

---

## 7. Configure .env

```env
# ── REQUIRED ──────────────────────────────────────────────────────────────────
DISCORD_TOKEN=your_bot_token_here
BOT_OWNER_ID=your_discord_user_id
GUILD_ID=your_server_id          # For instant command sync (recommended)

# ── CONTAINER ─────────────────────────────────────────────────────────────────
VPS_PREFIX=vps                   # Containers: vps-username-1
DEFAULT_STORAGE_POOL=default     # From: incus storage list

# ── RESOURCE LIMITS ───────────────────────────────────────────────────────────
MAX_RAM_GB=16
MAX_CPU_CORES=8
ENFORCE_RESOURCE_LIMITS=True

# ── MOONCOIN ──────────────────────────────────────────────────────────────────
MAIN_CHAT_ID=0                   # Channel ID where chatting earns coins

# ── DM COMMANDS ───────────────────────────────────────────────────────────────
DM_COMMANDS=False                # Enable via /system — then use /set to configure
DM_PREFIX=!                      # Prefix for DM commands when enabled

# ── EXTENSIONS (toggle via /system in Discord) ────────────────────────────────
ACCOUNT=True
MARKETPLACE=True
ISTS=False
MULTI_NODE=False
```

> **Role IDs are no longer set in .env.** Use `/set setting:Role` in Discord to map roles by ID — no restart needed.

---

## 8. Run the Bot

```bash
# Simple run
python3 bot.py

# With screen (stays alive after SSH disconnect)
screen -S moonbot
python3 bot.py
# Ctrl+A then D to detach
# screen -r moonbot to reattach

# With systemd (recommended for production)
cat > /etc/systemd/system/moonbot.service << 'EOF'
[Unit]
Description=MoonNodes VPS Bot
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/MoonNode
ExecStart=/usr/bin/python3 bot.py
Restart=always
RestartSec=10
Environment=PATH=/usr/bin:/snap/bin:/usr/local/bin

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable moonbot
systemctl start moonbot
systemctl status moonbot
journalctl -u moonbot -f
```

---

## 9. First Boot

On first start, the bot sends you (the owner) a DM:

```
👋 Welcome to MoonNodes!

Do you have an existing system.db from another MoonNodes bot?

[📤 Upload system.db]   [✅ Start Fresh]
```

**Upload system.db:**
- Click the button → bot waits for you to attach the `system.db` file in DM
- Bot downloads the file, replaces its database
- Bot scans all VPS records and recreates containers on the host
- All data is restored: VPS, coins, staff roles, settings, marketplace

**Start Fresh:**
- Bot starts with a clean database
- Use `/set` and `/role` to configure everything

---

## 10. First Steps in Discord

```bash
# 1. Set up channels
/set setting:Log Channel         value:CHANNEL_ID
/set setting:Ticket Category     value:CATEGORY_ID   # Right-click category → Copy ID
/set setting:Payment Channel     value:CHANNEL_ID
/set setting:Update Channel      value:CHANNEL_ID
/set setting:Claim Channel       value:CHANNEL_ID
/set setting:Main Chat Channel   value:CHANNEL_ID

# 2. Set role IDs (no .env needed)
/set setting:Role  role_action:add  permission:Admin  value:DISCORD_ROLE_ID
/set setting:Role  role_action:add  permission:Mod    value:DISCORD_ROLE_ID
/set setting:Role  role_action:list

# 3. Add staff
/role action:Add  user:@YourAdmin  permission:Admin

# 4. Create plans
/plans plan_type:Free     action:Add  plan_name:basic   ram_gb:1  cpu_cores:1  disk_gb:10  price:coin 100
/plans plan_type:Premium  action:Add  plan_name:starter ram_gb:2  cpu_cores:2  disk_gb:20  price:150

# 5. Add payment method
/set setting:Payment Method  payment_name:GCash  image:(attach QR image)

# 6. Enable extensions
/system action:Extension Toggle  extension:Ticket System       ext_state:Enable
/system action:Extension Toggle  extension:Account & MoonCoin  ext_state:Enable
/system action:Extension Toggle  extension:Marketplace         ext_state:Enable
```

---

## 11. Commands Reference

### 👤 User Commands

| Command | Description |
|---|---|
| `/help` | Role-based command guide with tabs |
| `/list` | Your VPS instances and live status |
| `/manage` | VPS control panel — Start, Stop, Restart, SSH, Console, Reinstall |
| `/status` | Host CPU, RAM, Disk stats |
| `/plans plan_type:` | Browse plans (Free / Premium / MoonCoin / Invite) |
| `/claim` | Claim a VPS with coins, invites, or payment proof |
| `/account` | Balance, VPS sharing, backups, upgrade info |
| `/support` | Open a ticket channel in the ticket category |
| `/marketplace` | Buy and sell VPS listings |

### 🌟 Staff Commands

| Command | Description |
|---|---|
| `/staff role:Staff action:My Tickets` | Your assigned tickets |
| `/staff role:Staff action:All Tickets` | All open ticket channels |
| `/staff role:Staff action:Close Ticket` | Delete ticket channel |
| `/staff role:Staff action:User Lookup` | Quick profile + VPS summary |
| `/staff role:Staff action:Add Note` | Add private staff note |
| `/staff role:Hard Staff action:User Info` | Full profile + coins + warnings |
| `/staff role:Hard Staff action:DM User` | Send staff DM |
| `/staff role:Hard Staff action:Warn User` | Issue a warning |
| `/staff role:Hard Staff action:Suspension Logs` | Recent suspensions |

### ⚒️ Mod Commands

| Command | Description |
|---|---|
| `/mod role:Mod action:User Info` | Full user profile |
| `/mod role:Mod action:VPS Info` | Container details |
| `/mod role:Mod action:Warn User` | Issue formal warning |
| `/mod role:Mod action:DM User` | Message a user as staff |
| `/mod role:Hard Mod action:Suspend VPS` | Force-suspend a container |
| `/mod role:Hard Mod action:Unsuspend VPS` | Re-enable a container |
| `/mod role:Hard Mod action:Blocklist` | Block user from all commands — `blocklist_action: add/remove/list` |
| `/mod role:Hard Mod action:Mute` | Mute user from /manage and DM commands — `mute_action: add/remove/list` |
| `/mod role:Hard Mod action:Lock Channel` | Lock a channel |
| `/mod role:Hard Mod action:Purge` | Delete messages (max 100) |

### 🛡️ Admin Commands

| Command | Description |
|---|---|
| `/create` | Deploy VPS for any user — node select → OS select |
| `/delete` | Permanently delete a VPS |
| `/manage user_id:` | Admin-view any user's VPS panel |
| `/list action:All VPS` | All VPS across every user |
| `/account admin_action:Coin` | Add/remove/view user MoonCoins |
| `/account admin_action:VPS` | Suspend/unsuspend/expiry |
| `/giveaway` | Start/cancel/list VPS giveaways |
| `/node` | Manage node cluster (List, Status, Create, Edit, Delete, Migrate) |
| `/role` | Assign / promote / demote / list staff |
| `/set` | Configure all bot settings |

### 👑 Owner Commands

| Command | Description |
|---|---|
| `/system action:Extension Toggle` | Enable/disable extensions |
| `/system action:Maintenance Mode` | Toggle maintenance + notify update channel |
| `/system action:💾 Database database_action:install` | Show restore instructions |
| `/system action:💾 Database database_action:delete` | Delete system.db (auto-backs up) |
| `/system action:Update Bot` | Pull from GitHub + unzip dev.zip |
| `/system action:MoonCoin Earning` | Manage coin earning rules |
| `/system action:Clean DB` | Remove stale VPS entries |

### ⚙️ Settings (`/set`)

| Setting | What it configures |
|---|---|
| `Log Channel` | Bot log channel |
| `Ticket Category` | Category where ticket channels are created |
| `Payment Channel` | Where payment proofs are sent |
| `Update Channel` | Maintenance mode announcements |
| `Claim Channel` | Where free claim node-select is posted |
| `Main Chat Channel` | Channel where chatting earns MoonCoins |
| `Payment Method` | Add/remove payment method images |
| `Block Terminal Command` | Block specific commands in VPS terminals |
| `User DM Commands` | Configure DM command prefix and allowed commands |
| `Role` | Map staff role names to Discord role IDs |

---

## 12. Extensions Reference

Toggle any extension at runtime:

```
/system action:Extension Toggle  extension:<Name>  ext_state:Enable|Disable
```

| Extension | What it does |
|---|---|
| **Multi-Node Support** | Deploy across LXD/Incus cluster; admin node-select for each claim |
| **Scheduled Backups** | Auto-snapshot all running containers on a schedule |
| **Ticket System** | `/support` creates dedicated channels in a ticket category |
| **Auto Expire & Suspend** | Automatically suspend VPS when expiry date passes |
| **Renewal Reminders** | DM users at 7, 3, and 1 day before expiry |
| **Bot 2FA** | Require a PIN for sensitive bot actions (delete VPS, wipe DB, etc.) |
| **Multi-Server Tickets** | Separate ticket categories per Discord server |
| **Backup Limits** | Cap maximum snapshots per VPS |
| **Marketplace** | Buy and sell VPS with coins or real money |
| **Account & MoonCoin** | `/account` command + full coin economy |
| **VPS Renewal** | Users can extend VPS expiry through account panel |
| **VPS Monitoring Alerts** | DM users when CPU or RAM exceeds threshold |
| **User Can Use Command in DM** | Allow bot commands in DMs using prefix `!` |
| **Block Terminal Commands** | Prevent specific shell commands in VPS terminals |

---

## 13. Transfer Data

All bot data is in `system.db`. Copying it moves everything.

```bash
# Stop bot on OLD server
systemctl stop moonbot

# Copy to new server
scp /root/MoonNode/system.db root@NEW_SERVER_IP:/root/MoonNode/system.db

# Start bot on NEW server
systemctl start moonbot
```

`system.db` contains: VPS records, MoonCoins, staff roles, settings, marketplace listings, warnings, notes, invite tracking, claim requests.

On next boot, all VPS containers that don't exist on the new host will be recreated automatically.

---

## 14. Update the Bot

```bash
# Make sure folder is a git repo first
cd /root/MoonNode
git init
git remote add origin https://github.com/MoonLink-Team/MoonNode
git fetch origin main
git reset --hard origin/main
```

Then from Discord:

```
/system action:Update Bot
```

The bot:
1. Stashes any local changes (`git stash`)
2. Pulls latest from GitHub (`git pull origin main`)
3. Unzips `dev.zip` if present (overwrites `.py` files, keeps `system.db` and `.env`)
4. Pops the stash

Then restart to apply:

```bash
systemctl restart moonbot
```

---

## 15. Troubleshooting

### "Invalid device type disk,path=…"

Fixed. The bot no longer uses `--device` inline. It runs `incus config device add` as a separate step after `incus init`.

### "No root device could be found"

The bot auto-detects your storage pool with `incus storage list`. If it still fails, set the pool explicitly:

```env
DEFAULT_STORAGE_POOL=default
```

And verify: `incus storage list`

### VPS has no internet after creation

The bot auto-detects bridges (`incusbr0`, `lxdbr0`) and attaches `eth0` automatically. If the VPS still has no internet after startup, run on the host:

```bash
# For Incus
incus network list                     # Check if incusbr0 exists
incus network create incusbr0          # Create if missing
incus config device add VPS_NAME eth0 nic nictype=bridged parent=incusbr0
incus restart VPS_NAME

# For LXD
lxc network create lxdbr0
lxc config device add VPS_NAME eth0 nic nictype=bridged parent=lxdbr0
lxc restart VPS_NAME
```

### "error: cannot set lxc.cgroup2.devices.allow"

Fixed. The bot now uses the correct no-space format: `lxc.cgroup2.devices.allow=a`. The error is also non-fatal and logged as a warning.

### "AttributeError: __enter__"

`data_lock` must be `asyncio.Lock`. Always use `async with data_lock:` — never `with data_lock:`. Fixed in all files.

### "/system Update Bot" — "cannot pull with rebase: unstaged changes"

Fixed. The bot now runs `git stash` before pulling and `git stash pop` after.

### Permission "Application did not respond"

Fixed. `CheckFailure` now calls `interaction.response.send_message` immediately (before any `defer`), so Discord always gets a response.

### "Synced 0 commands"

Check `GUILD_ID` in `.env`. Restart after changing it. For global sync (no GUILD_ID), commands take up to 1 hour to appear.

### Bot won't start — "No container runtime found"

```bash
which incus     # Should print /usr/bin/incus
which lxc       # Should print /snap/bin/lxc or /usr/bin/lxc

# Add snap to PATH if needed
echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc
source ~/.bashrc
```

---

*MoonNodes VPS Manager — built with discord.py + Incus/LXD*
