<p align="center">
  <h1 align="center">🌙 MoonNodes VPS Manager</h1>
  <p align="center">A Discord bot for managing LXC/LXD VPS instances — deploy, control, and monitor virtual servers directly from Discord slash commands.</p>
</p>

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Server Setup](#server-setup)
3. [Bot Setup on Discord Developer Portal](#bot-setup-on-discord-developer-portal)
4. [Installation](#installation)
5. [Configuration (.env)](#configuration-env)
6. [Running the Bot](#running-the-bot)
7. [First-Time Discord Setup](#first-time-discord-setup)
8. [All Commands](#all-commands)
9. [Extensions](#extensions)
10. [Setting Up a Node (Multi-Node Cluster)](#setting-up-a-node-multi-node-cluster)
11. [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Requirement | Version |
|---|---|
| OS | Ubuntu 20.04+ / Debian 11+ |
| Python | 3.10+ |
| LXD | 5.0+ |
| discord.py | 2.3+ |
| RAM (host) | 2 GB minimum |
| Storage | 20 GB minimum |

---

## Server Setup

### 1 — Update system
```bash
sudo apt update && sudo apt upgrade -y
```

### 2 — Install Python and pip
```bash
sudo apt install python3 python3-pip python3-venv git -y
python3 --version    # must be 3.10 or higher
```

### 3 — Install and initialize LXD
```bash
sudo apt install snapd -y
sudo snap install lxd
sudo lxd init --auto
# Verify:
lxc --version
lxc list
```

### 4 — (Optional) Create a dedicated storage pool
```bash
lxc storage create vpspool dir source=/var/lib/lxc/vpspool
lxc storage list    # note the pool name for DEFAULT_STORAGE_POOL in .env
```

### 5 — Clone the project
```bash
git clone https://github.com/MoonLink-Team/MoonNode
cd MoonNode
apt install zip
unzip dev.zip
```

### 6 — Create virtual environment and install dependencies
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
# or manually:
pip install discord.py python-dotenv aiohttp
```

### 7 — (Optional) Run as a systemd service
```bash
sudo nano /etc/systemd/system/moonnodes.service
```
Paste:
```ini
[Unit]
Description=MoonNodes VPS Discord Bot
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/moonnodes
ExecStart=/home/ubuntu/moonnodes/venv/bin/python bot.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
Enable:
```bash
sudo systemctl daemon-reload
sudo systemctl enable moonnodes
sudo systemctl start moonnodes
sudo systemctl status moonnodes    # should say "active (running)"
```

Useful service commands:
```bash
sudo systemctl restart moonnodes        # restart the bot
sudo systemctl stop moonnodes           # stop the bot
sudo journalctl -u moonnodes -f         # live logs
sudo journalctl -u moonnodes -n 100     # last 100 lines
```

---

## Bot Setup on Discord Developer Portal

1. Go to **https://discord.com/developers/applications**
2. Click **New Application** → give it a name
3. Go to **Bot** tab → **Add Bot**
4. Under **Privileged Gateway Intents**, enable:
   - ✅ Server Members Intent
   - ✅ Message Content Intent
5. Copy your **Bot Token** (used in `.env`)
6. Go to **OAuth2 → URL Generator**:
   - Scopes: `bot` + `applications.commands`
   - Bot Permissions: `Administrator`
7. Open the generated URL and invite the bot to your server

---

## Installation

```bash
cp .env.example .env
nano .env    # fill in the values below
```

---

## Configuration (.env)

```env
# ─────────────────────────────────────────────────────────────
# REQUIRED
# ─────────────────────────────────────────────────────────────
DISCORD_TOKEN=your_bot_token_here
BOT_OWNER_ID=your_discord_user_id_here     # right-click yourself → Copy User ID

# ─────────────────────────────────────────────────────────────
# APPEARANCE
# ─────────────────────────────────────────────────────────────
BOT_COLOR=5865F2                            # Hex embed colour (no #)

# ─────────────────────────────────────────────────────────────
# LXC / VPS
# ─────────────────────────────────────────────────────────────
VPS_PREFIX=vps                              # Prefix for container names
DEFAULT_STORAGE_POOL=default                # LXD pool name  (lxc storage list)
VPS_USER_ROLE_ID=0                          # Discord role ID granted after VPS creation

# ─────────────────────────────────────────────────────────────
# RESOURCE LIMITS
# ─────────────────────────────────────────────────────────────
MAX_RAM_GB=16
MAX_CPU_CORES=8
CPU_THRESHOLD=90                            # % usage that triggers a log warning
RAM_THRESHOLD=90

# ─────────────────────────────────────────────────────────────
# PLANS
# ─────────────────────────────────────────────────────────────
PLANS=True
CLAIM_PREMIUM=True
CLAIM_FREE=True

# ─────────────────────────────────────────────────────────────
# EXTENSIONS  (True / False)
# ─────────────────────────────────────────────────────────────
SOSB=False          # Scheduled Backups
ISTS=False          # Ticket System (support tickets)
AES=False           # Auto Expire & Suspend
RR=False            # Renewal Reminders
A2FA=False          # Admin 2-Factor PIN
MSTC=False          # Multi-Server Ticket Channels
UBL_ENABLED=False   # Backup Limits
BACKUP_LIMIT=1      # Default per-VPS backup cap
Multi-Node=False    # Multi-node LXD cluster

# ─────────────────────────────────────────────────────────────
# NEW FEATURES
# ─────────────────────────────────────────────────────────────
MARKETPLACE=True        # Enable /marketplace command
ACCOUNT=True            # Enable /account and MoonCoin system
MAIN_CHAT_ID=0          # Channel ID where chat gives +1 coin (0 = disabled)

# ─────────────────────────────────────────────────────────────
# CLOUDFLARE  (optional — automatic subdomain creation)
# ─────────────────────────────────────────────────────────────
CF_API_TOKEN=
CF_ZONE_ID=
CF_DOMAIN=yourdomain.com
HOST_PUBLIC_IP=
```

---

## Running the Bot

### Manually
```bash
source venv/bin/activate
python bot.py
```

### With systemd
```bash
sudo systemctl start moonnodes
```

On startup the bot will:
- Initialize the SQLite database (`moonnodes.db`)
- Load all command cogs
- Sync slash commands globally with Discord (takes up to 1 hour to appear everywhere, instant in your server if you use guild sync)

---

## First-Time Discord Setup

Run these as the **Bot Owner** after the bot is online:

### 1. Set channels
```
/set setting:Log Channel       value:<channel_id>
/set setting:Ticket Channel    value:<channel_id>
/set setting:Payment Channel   value:<channel_id>
```

### 2. Set resource alert thresholds
```
/set setting:CPU Threshold %   value:90
/set setting:RAM Threshold %   value:90
```

### 3. Add staff
```
/role action:Add permission:Owner  user:@yourself
/role action:Add permission:Admin  user:@youradmin
```

### 4. Add payment methods
```
/payment action:Add/Edit  payment_name:GCash   image:(attach QR screenshot)
/payment action:Add/Edit  payment_name:PayPal  image:(attach QR screenshot)
```

### 5. Add plans
```
# Free plans
/plans plan_type:Free  action:Add (Admin)  plan_name:basic    ram_gb:1  cpu_cores:1  disk_gb:10   price:Free
/plans plan_type:Free  action:Add (Admin)  plan_name:invite4  ram_gb:2  cpu_cores:1  disk_gb:15   price:invite 4

# Premium plans
/plans plan_type:Premium  action:Add (Admin)  plan_name:starter   ram_gb:2  cpu_cores:2  disk_gb:20  price:₱150  validity:30 Days
/plans plan_type:Premium  action:Add (Admin)  plan_name:pro        ram_gb:4  cpu_cores:4  disk_gb:40  price:₱300  validity:30 Days
/plans plan_type:Premium  action:Add (Admin)  plan_name:ultimate   ram_gb:8  cpu_cores:8  disk_gb:80  price:₱500  validity:30 Days
```

### 6. (Optional) Register additional nodes
```
/node create name:node-us-1 host:192.168.1.10 port:8443
/node list
```

---

## All Commands

### 👤 User Commands

| Command | Description |
|---|---|
| `/help` | Show all available commands |
| `/list` | View all your active VPS instances (status, specs, OS, expiry) |
| `/manage` | Open the VPS control panel. Tabs: **My VPS** and **Shared VPS** |
| `/backup action: vps_number:` | Create a snapshot, list backups, or restore the latest backup |
| `/share` | Grant another Discord user access to your VPS |
| `/plans` | Browse Free & Premium VPS plans with specs and pricing |
| `/claim` | Claim a plan by name or redeem a promo code |
| `/transfer` | Sell or gift your VPS to another user |
| `/upgrade` | Upgrade your VPS plan using credits |
| `/support vps_number: issue:` | Open a private support thread for a specific VPS |
| `/status` | View live CPU, RAM, and disk usage of the host node |
| `/affiliate` | View referral credits or transfer credits to another user |

---

### /manage — Control Panel Detail

When you run `/manage` (no arguments), you see two tabs:

**🖥️ My VPS** — Manage your own VPS instances.
Actions available:
- ▶️ **Start** — Boot the container
- ⏸️ **Stop** — Gracefully stop the container
- 🔄 **Reinstall OS** — Wipe and reinstall from scratch (confirmation required)
- 🔑 **Web SSH** — Generate a tmate SSH link sent to your DMs
- 💻 **Console Logs** — View the last 15 lines of system logs
- 💿 **Install Software** — One-click installer for Docker, Node.js, Nginx, Python, Java, Minecraft, Rust, CS2, FiveM, and more
- 🔄 **Refresh Stats** — Refresh live CPU / RAM / Disk stats

**🤝 Shared VPS** — Manage VPS that other users have shared with you.
(Start, Stop, Web SSH, Console, Software Install — no Reinstall)

Admins can also run `/manage user:@someone` to manage any user's VPS.

---

### 🛡️ Admin / Owner Commands

| Command | Description |
|---|---|
| `/role action: permission: user:` | Assign or remove staff roles (Owner / Admin / Mod / Dev) |
| `/set setting: value: [option:] [date:] [time:]` | Configure channels, thresholds, backup schedules |
| `/payment action: payment_name: [image:]` | Add, edit, remove, or list payment methods |
| `/create user: plan_name: plan_type:` | Deploy a new VPS (select node → select OS) |
| `/delete user: vps_number: [reason:]` | Permanently delete a VPS container |
| `/vps action: container_name: [reason:]` | Suspend or unsuspend a VPS |
| `/list-all` | View all VPS across every user and node |
| `/giveaway action:` | Start, cancel, or list active giveaways |
| `/affiliate` | View, add, or remove referral credits for any user |
| `/system-admin action:` | Extension toggles, maintenance mode, DB clean, A2FA |
| `/lxc-list [node_id]` | List raw LXC containers on a node |

---

### 🌐 Node Management (`/node`)

| Command | Description |
|---|---|
| `/node list` | List all registered nodes with host/port |
| `/node status [node_id]` | Live CPU / RAM / Disk stats per node (blank = all nodes) |
| `/node create name: host: [port:]` | Register a new LXD remote and auto-configure the remote |
| `/node edit node_id: [new_name:] [new_host:] [new_port:]` | Update node connection details |
| `/node delete node_id:` | Remove a node (only if no VPS are on it) |
| `/node migrate vps_name: to_node:` | Live-migrate a VPS container to another node |
| `/lxc-list [node_id]` | Show raw `lxc list` output for any registered node |

---

### ⚙️ /set Settings Reference

| Setting | Value Format | Description |
|---|---|---|
| `Log Channel` | Channel ID | Where bot activity logs are sent |
| `Payment Channel` | Channel ID | Where payment proof screenshots go |
| `Ticket Channel` | Channel ID | Where support threads are created |
| `CPU Threshold %` | Number (0–100) | CPU % that triggers a warning log |
| `RAM Threshold %` | Number (0–100) | RAM % that triggers a warning log |
| `Backup Limits` | `enable` / `disable` / number | Max snapshots per VPS (requires Backup Limits extension) |
| `Scheduled Backups` | `enable` / `disable` | Toggle scheduled auto-snapshots |

For `Backup Limits` and `Scheduled Backups` you can also pass:
- `option:` — frequency or per-plan cap (e.g. `daily`, `weekly`, `3`)
- `date:` — start date in `1/12/2025` format
- `time:` — run time in `12:00AM` format

Example:
```
/set setting:Scheduled Backups  value:enable  option:daily  date:1/1/2025  time:2:00AM
```

---

### ⚙️ /system-admin Actions Reference

| Action | Description |
|---|---|
| `Extension Toggle` | Enable or disable an extension at runtime |
| `Maintenance Mode` | Toggle maintenance mode (bot shows "Under Maintenance" status) |
| `View Extensions` | See all extensions and their current On/Off state |
| `Clean DB` | Remove stale/empty entries from the VPS database |
| `A2FA Management` | Set or remove your admin 2-Factor PIN |

---

## Extensions

Extensions are optional features you can toggle on/off from `/system-admin action:Extension Toggle` or by setting them in `.env`.

| Extension | .env Key | What it does |
|---|---|---|
| **Multi-Node Support** | `Multi-Node=True` | Enables multi-server LXD cluster. Register additional nodes with `/node create`. VPS can be created on or migrated to any node. |
| **Scheduled Backups** | `SOSB=True` | Automatically snapshots all running VPS every 24 hours. Schedule the time with `/set setting:Scheduled Backups`. |
| **Ticket System** | `ISTS=True` | Enables `/support` to open private Discord threads for VPS issues. Requires a Ticket Channel set via `/set`. |
| **Auto Expire & Suspend** | `AES=True` | Automatically stops a VPS when its expiry date passes. Notifies the owner by DM. |
| **Renewal Reminders** | `RR=True` | Sends a DM to the VPS owner 3 days before their VPS expires, reminding them to renew. |
| **Admin 2FA** | `A2FA=True` | Requires a 4-digit PIN for sensitive admin actions. Set your PIN with `/system-admin action:A2FA Management`. |
| **Multi-Server Tickets** | `MSTC=True` | Allows support tickets to be routed to different channels per server (for multi-guild setups). |
| **Backup Limits** | `UBL_ENABLED=True` | Enforces a maximum number of snapshots per VPS. Configure the cap with `/set setting:Backup Limits`. |

---

## Setting Up a Node (Multi-Node Cluster)

A **node** is any Linux server running LXD that the bot can connect to remotely. Your main bot server is always `local` (Node #1). You can add as many extra servers as you want — each becomes a separate node that can host VPS containers independently.

```
┌─────────────────────────────────────────────────────────┐
│                   Discord Bot (local)                   │
│               bot.py  ·  moonnodes.db                   │
└──────────────┬──────────────────────────┬───────────────┘
               │  LXD API (port 8443)     │  LXD API (port 8443)
               ▼                          ▼
   ┌───────────────────┐      ┌───────────────────┐
   │   Node: node-sg-1 │      │   Node: node-us-1 │
   │   Singapore VPS   │      │   US East VPS     │
   └───────────────────┘      └───────────────────┘
```

---

### Step 1 — Requirements for the Remote Node Server

Each node server needs:

| Requirement | Details |
|---|---|
| OS | Ubuntu 20.04+ or Debian 11+ |
| LXD | 5.0+ (installed via snap) |
| Open port | **8443** accessible from the bot server |
| RAM | 2 GB minimum (more depending on how many VPS you host) |
| Storage | 20 GB minimum |

The remote node does **not** need Python or the bot code. It only needs LXD running.

---

### Step 2 — Install Incus or LXD on the Remote Node

You can use either **Incus** (recommended) or **LXD**. The bot auto-detects which one is installed — you don't need to configure anything.

### Step 2a — Install via Incus (Recommended — Zabbly repo)

Incus is the modern successor to LXD. The bot detects it automatically.

```bash
# 1. Install dependencies
sudo apt update
sudo apt install curl -y
sudo mkdir -p /etc/apt/keyrings/

# 2. Add Zabbly repository key
curl -fsSL https://pkgs.zabbly.com/key.asc | sudo tee /etc/apt/keyrings/zabbly.asc > /dev/null

# 3. Register the Incus stable repo
echo "deb [signed-by=/etc/apt/keyrings/zabbly.asc] https://pkgs.zabbly.com/incus/stable $(. /etc/os-release && echo ${VERSION_CODENAME}) main" | \
  sudo tee /etc/apt/sources.list.d/zabbly-incus-stable.list

# 4. Install Incus
sudo apt update
sudo apt install incus -y

# 5. Initialize
sudo incus admin init --auto

# 6. (Optional) run without sudo
sudo usermod -aG incus-admin $USER
newgrp incus-admin

# 7. Verify
incus list
```

> Use `incus` everywhere you'd normally type `lxc`. The bot handles this automatically.

---

### Step 2b — Install via LXD / Snap (Alternative)



SSH into the remote server and run:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install snapd if not present
sudo apt install snapd -y

# Install LXD
sudo snap install lxd

# Add your user to the lxd group
sudo usermod -aG lxd $USER
newgrp lxd

# Verify
lxc --version
```

---

### Step 3 — Initialize LXD on the Remote Node

Run the LXD setup wizard:

```bash
sudo lxd init
```

When prompted, use these recommended answers:

```
Would you like to use LXD clustering? (yes/no) [default=no]: no
Do you want to configure a new storage pool? (yes/no) [default=yes]: yes
Name of the new storage pool [default=default]: default
Name of the storage backend to use (btrfs, dir, lvm, zfs, ceph) [default=zfs]: dir
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to create a new local network bridge? (yes/no) [default=yes]: yes
What should the new bridge be called? [default=lxdbr0]: lxdbr0
What IPv4 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]: auto
What IPv6 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]: none
Would you like the LXD server to be available over the network? (yes/no) [default=no]: YES  ← important
Address to bind LXD to (not including port) [default=all]: all
Port to bind LXD to [default=8443]: 8443
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: yes
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: no
```

> **Critical:** Answer **yes** to "Would you like the LXD server to be available over the network?" — this is what allows the bot server to connect to this node.

---

### Step 4 — Open Port 8443 on the Remote Node

```bash
# Using UFW (Ubuntu default firewall)
sudo ufw allow 8443/tcp
sudo ufw reload
sudo ufw status

# Or using iptables directly
sudo iptables -A INPUT -p tcp --dport 8443 -j ACCEPT
```

If your server is behind a cloud provider firewall (AWS, GCP, DigitalOcean, Hetzner, etc.), also open port **8443** in the provider's security group / firewall rules panel.

Verify it's reachable from the bot server:
```bash
# Run this on the BOT server, not the node
nc -zv <node-ip> 8443
# Should say: Connection to <node-ip> 8443 port [tcp/*] succeeded!
```

---

### Step 5 — Trust the Bot Server on the Node

On the **remote node server**, generate and show the LXD trust certificate token:

```bash
# LXD 5.x — use tokens (recommended)
lxc config trust add bot-server --name moonnodes
```

This outputs a one-time token. Copy it — you'll use it in Step 6.

**Alternative (password-based, older LXD versions):**
```bash
lxc config set core.trust_password your-secret-password
```

---

### Step 6 — Add the Remote Node as an LXD Remote on the Bot Server

On the **bot server** (where `bot.py` runs):

**Token method (LXD 5.x):**
```bash
lxc remote add node-sg-1 https://<node-ip>:8443 --token <paste-token-here>
```

**Password method (older LXD):**
```bash
lxc remote add node-sg-1 https://<node-ip>:8443 --accept-certificate
# Enter the trust password when prompted
```

**Verify the connection:**
```bash
lxc remote list
# node-sg-1 should appear

lxc list node-sg-1:
# Should show (empty list) with no errors
```

> Replace `node-sg-1` with your chosen name. Use the same name when registering in the bot.

---

### Step 7 — Enable Multi-Node in .env

On the bot server, edit your `.env`:

```env
Multi-Node=True
```

Then restart the bot:
```bash
sudo systemctl restart moonnodes
# or:
source venv/bin/activate && python bot.py
```

---

### Step 8 — Register the Node in the Bot

In Discord, run:

```
/node create  name:node-sg-1  host:<nodely delete a VPS container |
| `/vps action: container_name: [reason:]` | Suspend or unsuspend a VPS |
| `/list-all` | View all VPS across every user and node |
| `/giveaway action:` | Start, cancel, or list active giveaways |
| `/affiliate` | View, add, or remove referral credits for any user |
| `/system-admin action:` | Extension toggles, maintenance mode, DB clean, A2FA |
| `/lxc-list [node_id]` | List raw LXC containers on a node |

---

### 🌐 Node Management (`/node`)

| Command | Description |
|---|---|
| `/node list` | List all registered nodes with host/port |
| `/node status [node_id]` | Live CPU / RAM / Disk stats per node (blank = all nodes) |
| `/node create name: host: [port:]` | Register a new LXD remote and auto-configure the remote |
| `/node edit node_id: [new_name:] [new_host:] [new_port:]` | Update node connection details |
| `/node delete node_id:` | Remove a node (only if no VPS are on it) |
| `/node migrate vps_name: to_node:` | Live-migrate a VPS container to another node |
| `/lxc-list [node_id]` | Show raw `lxc list` output for any registered node |

---

### ⚙️ /set Settings Reference

| Setting | Value Format | Description |
|---|---|---|
| `Log Channel` | Channel ID | Where bot activity logs are sent |
| `Payment Channel` | Channel ID | Where payment proof screenshots go |
| `Ticket Channel` | Channel ID | Where support threads are created |
| `CPU Threshold %` | Number (0–100) | CPU % that triggers a warning log |
| `RAM Threshold %` | Number (0–100) | RAM % that triggers a warning log |
| `Backup Limits` | `enable` / `disable` / number | Max snapshots per VPS (requires Backup Limits extension) |
| `Scheduled Backups` | `enable` / `disable` | Toggle scheduled auto-snapshots |

For `Backup Limits` and `Scheduled Backups` you can also pass:
- `option:` — frequency or per-plan cap (e.g. `daily`, `weekly`, `3`)
- `date:` — start date in `1/12/2025` format
- `time:` — run time in `12:00AM` format

Example:
```
/set setting:Scheduled Backups  value:enable  option:daily  date:1/1/2025  time:2:00AM
```

---

### ⚙️ /system-admin Actions Reference

| Action | Description |
|---|---|
| `Extension Toggle` | Enable or disable an extension at runtime |
| `Maintenance Mode` | Toggle maintenance mode (bot shows "Under Maintenance" status) |
| `View Extensions` | See all extensions and their current On/Off state |
| `Clean DB` | Remove stale/empty entries from the VPS database |
| `A2FA Management` | Set or remove your admin 2-Factor PIN |

---

## Extensions

Extensions are optional features you can toggle on/off from `/system-admin action:Extension Toggle` or by setting them in `.env`.

| Extension | .env Key | What it does |
|---|---|---|
| **Multi-Node Support** | `Multi-Node=True` | Enables multi-server LXD cluster. Register additional nodes with `/node create`. VPS can be created on or migrated to any node. |
| **Scheduled Backups** | `SOSB=True` | Automatically snapshots all running VPS every 24 hours. Schedule the time with `/set setting:Scheduled Backups`. |
| **Ticket System** | `ISTS=True` | Enables `/support` to open private Discord threads for VPS issues. Requires a Ticket Channel set via `/set`. |
| **Auto Expire & Suspend** | `AES=True` | Automatically stops a VPS when its expiry date passes. Notifies the owner by DM. |
| **Renewal Reminders** | `RR=True` | Sends a DM to the VPS owner 3 days before their VPS expires, reminding them to renew. |
| **Admin 2FA** | `A2FA=True` | Requires a 4-digit PIN for sensitive admin actions. Set your PIN with `/system-admin action:A2FA Management`. |
| **Multi-Server Tickets** | `MSTC=True` | Allows support tickets to be routed to different channels per server (for multi-guild setups). |
| **Backup Limits** | `UBL_ENABLED=True` | Enforces a maximum number of snapshots per VPS. Configure the cap with `/set setting:Backup Limits`. |

---

## Setting Up a Node (Multi-Node Cluster)

A **node** is any Linux server running LXD that the bot can connect to remotely. Your main bot server is always `local` (Node #1). You can add as many extra servers as you want — each becomes a separate node that can host VPS containers independently.

```
┌─────────────────────────────────────────────────────────┐
│                   Discord Bot (local)                   │
│               bot.py  ·  moonnodes.db                   │
└──────────────┬──────────────────────────┬───────────────┘
               │  LXD API (port 8443)     │  LXD API (port 8443)
               ▼                          ▼
   ┌───────────────────┐      ┌───────────────────┐
   │   Node: node-sg-1 │      │   Node: node-us-1 │
   │   Singapore VPS   │      │   US East VPS     │
   └───────────────────┘      └───────────────────┘
```

---

### Step 1 — Requirements for the Remote Node Server

Each node server needs:

| Requirement | Details |
|---|---|
| OS | Ubuntu 20.04+ or Debian 11+ |
| LXD | 5.0+ (installed via snap) |
| Open port | **8443** accessible from the bot server |
| RAM | 2 GB minimum (more depending on how many VPS you host) |
| Storage | 20 GB minimum |

The remote node does **not** need Python or the bot code. It only needs LXD running.

---

### Step 2 — Install Incus or LXD on the Remote Node

You can use either **Incus** (recommended) or **LXD**. The bot auto-detects which one is installed — you don't need to configure anything.

### Step 2a — Install via Incus (Recommended — Zabbly repo)

Incus is the modern successor to LXD. The bot detects it automatically.

```bash
# 1. Install dependencies
sudo apt update
sudo apt install curl -y
sudo mkdir -p /etc/apt/keyrings/

# 2. Add Zabbly repository key
curl -fsSL https://pkgs.zabbly.com/key.asc | sudo tee /etc/apt/keyrings/zabbly.asc > /dev/null

# 3. Register the Incus stable repo
echo "deb [signed-by=/etc/apt/keyrings/zabbly.asc] https://pkgs.zabbly.com/incus/stable $(. /etc/os-release && echo ${VERSION_CODENAME}) main" | \
  sudo tee /etc/apt/sources.list.d/zabbly-incus-stable.list

# 4. Install Incus
sudo apt update
sudo apt install incus -y

# 5. Initialize
sudo incus admin init --auto

# 6. (Optional) run without sudo
sudo usermod -aG incus-admin $USER
newgrp incus-admin

# 7. Verify
incus list
```

> Use `incus` everywhere you'd normally type `lxc`. The bot handles this automatically.

---

### Step 2b — Install via LXD / Snap (Alternative)



SSH into the remote server and run:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install snapd if not present
sudo apt install snapd -y

# Install LXD
sudo snap install lxd

# Add your user to the lxd group
sudo usermod -aG lxd $USER
newgrp lxd

# Verify
lxc --version
```

---

### Step 3 — Initialize LXD on the Remote Node

Run the LXD setup wizard:

```bash
sudo lxd init
```

When prompted, use these recommended answers:

```
Would you like to use LXD clustering? (yes/no) [default=no]: no
Do you want to configure a new storage pool? (yes/no) [default=yes]: yes
Name of the new storage pool [default=default]: default
Name of the storage backend to use (btrfs, dir, lvm, zfs, ceph) [default=zfs]: dir
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to create a new local network bridge? (yes/no) [default=yes]: yes
What should the new bridge be called? [default=lxdbr0]: lxdbr0
What IPv4 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]: auto
What IPv6 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]: none
Would you like the LXD server to be available over the network? (yes/no) [default=no]: YES  ← important
Address to bind LXD to (not including port) [default=all]: all
Port to bind LXD to [default=8443]: 8443
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: yes
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: no
```

> **Critical:** Answer **yes** to "Would you like the LXD server to be available over the network?" — this is what allows the bot server to connect to this node.

---

### Step 4 — Open Port 8443 on the Remote Node

```bash
# Using UFW (Ubuntu default firewall)
sudo ufw allow 8443/tcp
sudo ufw reload
sudo ufw status

# Or using iptables directly
sudo iptables -A INPUT -p tcp --dport 8443 -j ACCEPT
```

If your server is behind a cloud provider firewall (AWS, GCP, DigitalOcean, Hetzner, etc.), also open port **8443** in the provider's security group / firewall rules panel.

Verify it's reachable from the bot server:
```bash
# Run this on the BOT server, not the node
nc -zv <node-ip> 8443
# Should say: Connection to <node-ip> 8443 port [tcp/*] succeeded!
```

---

### Step 5 — Trust the Bot Server on the Node

On the **remote node server**, generate and show the LXD trust certificate token:

```bash
# LXD 5.x — use tokens (recommended)
lxc config trust add bot-server --name moonnodes
```

This outputs a one-time token. Copy it — you'll use it in Step 6.

**Alternative (password-based, older LXD versions):**
```bash
lxc config set core.trust_password your-secret-password
```

---

### Step 6 — Add the Remote Node as an LXD Remote on the Bot Server

On the **bot server** (where `bot.py` runs):

**Token method (LXD 5.x):**
```bash
lxc remote add node-sg-1 https://<node-ip>:8443 --token <paste-token-here>
```

**Password method (older LXD):**
```bash
lxc remote add node-sg-1 https://<node-ip>:8443 --accept-certificate
# Enter the trust password when prompted
```

**Verify the connection:**
```bash
lxc remote list
# node-sg-1 should appear

lxc list node-sg-1:
# Should show (empty list) with no errors
```

> Replace `node-sg-1` with your chosen name. Use the same name when registering in the bot.

---

### Step 7 — Enable Multi-Node in .env

On the bot server, edit your `.env`:

```env
Multi-Node=True
```

Then restart the bot:
```bash
sudo systemctl restart moonnodes
# or:
source venv/bin/activate && python bot.py
```

---

### Step 8 — Register the Node in the Bot

In Discord, run:

```
/node create  name:node-sg-1  host:<node-ip>  port:8443
```

Use the **exact same name** you used in Step 6 when running `lxc remote add`.

The bot will:
1. Save the node to the database
2. Attempt to auto-add the LXD remote (if not already added in Step 6)
3. Report success or any error

Verify it registered correctly:
```
/node list
/node status node_id:2
```

---

### Step 9 — Create a Storage Pool on the Node (Optional but Recommended)

On the **remote node server**:

```bash
# Using directory-based pool (simplest)
lxc storage create vpspool dir source=/var/lib/lxc/vpspool

# Using ZFS (better performance, requires zfs-utils)
sudo apt install zfsutils-linux -y
lxc storage create vpspool zfs

# Verify
lxc storage list
```

When creating VPS on this node via `/create`, the bot will use the pool configured in the node.

---

### Step 10 — Migrate an Existing VPS to the New Node

If you want to move a VPS from `local` to the new node:

```
/node migrate  vps_name:vps-username-1  to_node:node-sg-1
```

The bot will:
1. Stop the container on the source node
2. Move it via `lxc move` to the destination node
3. Start it on the new node
4. Update the database

---

### Quick Reference — Node Setup Checklist

```
Remote Node Server:
  ☐ Ubuntu 20.04+ / Debian 11+ installed
  ☐ LXD installed via snap
  ☐ lxd init run with "available over network: YES"
  ☐ Port 8443 open in UFW
  ☐ Port 8443 open in cloud firewall (if applicable)
  ☐ Trust token generated: lxc config trust add bot-server

Bot Server:
  ☐ lxc remote add <name> https://<ip>:8443 --token <token>
  ☐ lxc list <name>:  returns without error
  ☐ Multi-Node=True set in .env
  ☐ Bot restarted

Discord:
  ☐ /node create name:<name> host:<ip> port:8443
  ☐ /node list  shows the new node
  ☐ /node status  shows live stats for the new node
```

---

### Troubleshooting Nodes

**`connection refused` when adding remote**
Port 8443 is not open. Check UFW and cloud firewall rules on the node server.

**`certificate verify failed`**
Use `--accept-certificate` flag:
```bash
lxc remote add node-sg-1 https://<ip>:8443 --accept-certificate
```

**`authentication failure`**
The trust password or token was wrong or expired. Generate a new token on the node:
```bash
lxc config trust add bot-server --name moonnodes
```

**Node shows as offline in `/node status`**
- Check LXD is running on the node: `sudo snap services lxd`
- Restart LXD if needed: `sudo snap restart lxd`
- Check the bot server can still reach port 8443: `nc -zv <ip> 8443`

**`lxc move` fails during migration**
Both nodes must be able to reach each other directly (not just from the bot server). Check network connectivity between nodes:
```bash
# On node 1, try to reach node 2
nc -zv <node2-ip> 8443
```

---


---

## Marketplace & MoonCoin System

### /marketplace

The marketplace lets users sell their VPS instances to each other using MoonCoins, Invites, or Real Money.

Enable it in `.env`:
```env
MARKETPLACE=True
```

Or toggle at runtime:
```
/set setting:Marketplace value:enable
```

| Action | Command | Description |
|---|---|---|
| Browse | `/marketplace pick_one:List` | See all active VPS listings |
| Sell | `/marketplace pick_one:Sell sell_name: vps_number: price: price_type:` | List your VPS for sale |
| Buy | `/marketplace pick_one:Buy sell_name: user:` | Purchase a listing |
| Edit | `/marketplace pick_one:Edit sell_name: new_price: new_name:` | Update your listing |
| Remove | `/marketplace pick_one:Remove sell_name:` | Remove your listing |
| Payment | `/marketplace pick_one:Payment pay_action: payment_name: image:` | Admin: manage payment methods |

**Price types:** `coin` (🌙 MoonCoins) · `inv` (📨 Invites) · `real` (💵 Real Money)

Real Money purchases open a support ticket flow for admin-assisted transfer.

---

### /account — MoonCoin System

MoonCoin (🌙) is the in-server currency used to claim free VPS plans and buy marketplace listings.

Enable it in `.env`:
```env
ACCOUNT=True
MAIN_CHAT_ID=<your-main-channel-id>   # channel that gives +1 coin per message
```

**Earning MoonCoins:**

| Action | Reward | Notes |
|---|---|---|
| Invite someone to the server | +5 coins | Tracked via Discord invite API |
| Chat in the main channel | +1 coin | 5 minute cooldown per user |
| Join a giveaway | +3 coins | Awarded when giveaway is joined |
| Join an event | +7 coins | Awarded when event is joined |
| Have VIP role | +1% multiplier | Applied to all coin earnings |
| 1st server boost | +2% multiplier | Permanent for that account |
| 2nd server boost | +3% multiplier | Stacks with first boost |

**Commands:**

| Command | Description |
|---|---|
| `/account` | View your own balance and stats |
| `/account target_user:@user` | View another user's account |
| `/account admin_action:Add Coins transfer_amount:500 target_user:@user` | Admin: give coins |
| `/account admin_action:Remove Coins transfer_amount:100 target_user:@user` | Admin: remove coins |
| `/account admin_action:Transfer Coins transfer_amount:200 target_user:@from transfer_to:@to` | Admin: move coins between users |
| `/account admin_action:Reset Account target_user:@user` | Admin: reset balance to zero |

**Spending MoonCoins:**
- `/claim claim_type:Free option:coin code_or_plan:basic` — claim a free VPS plan
- `/marketplace pick_one:Buy` — purchase a VPS listing from another user


---

## Troubleshooting

### Slash commands not appearing
Discord can take up to 1 hour to propagate global slash commands. During development, use guild-specific sync:
```python
# In bot.py setup_hook, change:
synced = await self.tree.sync()
# To (replace YOUR_GUILD_ID):
guild = discord.Object(id=YOUR_GUILD_ID)
self.tree.copy_global_to(guild=guild)
synced = await self.tree.sync(guild=guild)
```

### "LXC command not found" on startup
```bash
sudo snap install lxd
sudo lxd init --auto
# Add your user to the lxd group:
sudo usermod -aG lxd $USER
newgrp lxd
```

### Web SSH (tmate) not working
The VPS needs internet access. Check:
```bash
lxc exec <container> -- ping -c 3 8.8.8.8
```
If it fails, check the host network bridge configuration:
```bash
lxc network list
lxc network show lxdbr0
```

### Bot token invalid
Regenerate your token in the Discord Developer Portal under **Bot → Reset Token** and update `.env`.

### Database errors
```bash
# Backup current DB
cp moonnodes.db moonnodes.db.bak
# Clean stale entries via Discord:
/system-admin action:Clean DB
```

### VPS stuck in "stopping"
```bash
lxc stop <container> --force
```

---

## File Structure

```
moonnodes/
├── bot.py                    # Entry point — loads cogs, starts tasks
├── .env                      # Your configuration (never commit this)
├── moonnodes.db              # SQLite database (auto-created)
├── core/
│   ├── config.py             # Loads all .env settings
│   ├── database.py           # DB helpers (init, load, save, queries)
│   ├── lxc.py                # LXC/LXD command wrappers and node stats
│   ├── monitoring.py         # Background resource monitor thread
│   ├── theme.py              # Embed factories and visual helpers
│   └── __init__.py           # Detects active storage pool on import
├── commands/
│   ├── Owner/                # Bot Owner commands
│   │   ├── admin.py          # /role
│   │   ├── create.py         # /create
│   │   ├── delete.py         # /delete
│   │   ├── giveaway.py       # /giveaway
│   │   ├── list_all.py       # /list-all
│   │   ├── nodes.py          # /node group + /lxc-list
│   │   └── system_admin.py   # /system-admin
│   ├── Admin/
│   │   ├── set_cmd.py        # /set + /payment
│   │   └── vps.py            # /vps
│   └── User/
│       ├── help.py           # /help
│       ├── list_cmd.py       # /list
│       ├── manage.py         # /manage (My VPS + Shared VPS tabs)
│       ├── plans.py          # /plans
│       ├── claim.py          # /claim
│       ├── backup.py         # /backup
│       ├── share.py          # /share
│       ├── affiliate.py      # /affiliate
│       ├── support.py        # /support
│       ├── status.py         # /status
│       ├── transfer.py       # /transfer
│       └── upgrade.py        # /upgrade
├── ui/
│   ├── manage_view.py        # ManageView interactive buttons
│   ├── os_select.py          # OS selection dropdown for /create
│   ├── node_select.py        # Node selection dropdown
│   ├── plans_view.py         # Plans embed UI
│   ├── software_view.py      # One-click software installer
│   └── confirm_view.py       # Generic confirm/cancel dialog
└── feature/
    ├── plans/                # Plan data management
    └── codes/                # Promo code management
```

---

*MoonNodes VPS Manager — Discord LXC/LXD VPS management bot*

---

## Changelog — Latest Updates

### Bug Fixes
- ✅ **`requirements.txt` created** — install with `pip install -r requirements.txt`
- ✅ **`/node` CommandAlreadyRegistered** — fixed duplicate tree registration
- ✅ **`/help` field too long (50035)** — all embed fields now auto-truncate at 1024 chars; `/help` is now a paginated 4-page view with buttons
- ✅ **LXD / Incus auto-detection** — bot now checks for `incus` first, falls back to `lxc` (LXD). All container commands route through the detected runtime automatically

### New Features
- 🆕 **`/marketplace`** — Buy, sell, edit, and browse VPS listings using MoonCoins, Invites, or Real Money
- 🆕 **`/account`** — MoonCoin balance, stats, earning info, and admin coin management
- 🆕 **MoonCoin system** — Passive earning via invites, chat, events, giveaways; multipliers for VIP/booster roles
- 🆕 **`/claim` inv/coin** — Free plans now accept Invites or MoonCoins as payment; balances are deducted on claim
- 🆕 **Incus support** — Full Incus container runtime support alongside LXD

### Changes
- 🔄 `/role list` — Premium UI with role icons, colors, member join dates
- 🔄 `/role add/remove` — Detailed DM sent to user with role-specific message
- 🔄 Free plans — Now show dual-currency prices (Invites + MoonCoins) in `/plans`
- 🔄 `/help` — Now a tabbed 4-page interactive menu (User · Admin · Settings · Node)
- 🗑️ Removed `/transfer`, `/affiliate`, `/claim Promo Code`
- 🗑️ `/system-admin` — Removed dead extensions: Advanced Server Monitoring, Economy System, Referral Affiliate, Promo Codes, Per-User Disk Cap

---

## Changelog — This Update

### Bug Fixes
- ✅ **`lxc -c` flag error** — Removed unsupported `-c <cols>` shorthand from all LXC/Incus commands. `--format=csv` is now used everywhere without column filters
- ✅ **Incus not working** — `core/lxc.py` now auto-detects `incus` or `lxc` at startup and routes every command through the correct binary. `_cmd()` and `_fix_args()` handle all edge cases
- ✅ **A2FA `a2fa_action` not working** — Fixed: now checks `cfg.A2FA` first, then handles `edit`/`remove` correctly with proper DB updates
- ✅ **PyNaCl / davey warnings** — Suppressed via `warnings.filterwarnings` and `logging.getLogger("discord.voice_client").setLevel(ERROR)`
- ✅ **`Command failed (no output)`** — `execute_lxc` now logs both stdout and stderr in all failure cases

### New Commands & Features
- 🔄 **`/node`** — Converted from 6 separate subcommands to **one command with `pick_one:` choices**. Create and Edit use Discord Modals (popup forms) — no parameters needed in the command bar
- 🎭 **`/help`** — Fully role-based pages. Users see User commands only. Admins see Admin panel. Mods see Mod tools. Devs see Dev tools. Owners see everything. Tab buttons switch between pages
- 🎫 **`/support`** — Now gated behind ISTS extension. Posts **Claim / Close / Mark Done** buttons inside the thread. DMs the ticket opener when claimed or closed
- 🏆 **Giveaway winner DM** — Rich premium embed with container name, specs, OS, and instructions
- 🛒 **Marketplace buy** — Container is renamed to reflect new owner. Seller receives a premium DM

### Extensions — All Now Working

| Extension | Task | Frequency |
|---|---|---|
| **Scheduled Backups** (SOSB) | Auto-snapshots all running VPS | Hourly check, runs at configured time |
| **Auto Expire & Suspend** (AES) | Stops VPS past expiry, DMs owner | Every 6 hours |
| **Renewal Reminders** (RR) | DMs owner at 7, 3, 1 day before expiry | Every 6 hours |
| **VPS Monitoring Alerts** | DMs owner when CPU/RAM exceeds threshold | Every 10 minutes |
| **Backup Limits** (UBL) | Deletes oldest auto-snapshots over the limit | Every 12 hours |
| **Ticket System** (ISTS) | `/support` enabled, Close/Claim/Done buttons | On demand |
| **Multi-Server Tickets** (MSTC) | Verifies ticket channels exist per guild | Every 24 hours |
| **Admin 2FA** (A2FA) | PIN modal blocks sensitive actions | On each admin action |
| **Multi-Node** | Enables multi-node cluster commands | On demand |

### Extension List (all toggleable in `/system-admin`)
Multi-Node Support · Scheduled Backups · Ticket System · Auto Expire & Suspend · Renewal Reminders · Admin 2FA · Multi-Server Tickets · Backup Limits · Marketplace · Account & MoonCoin · VPS Renewal · VPS Monitoring Alerts · VPS Templates · Multi-language


---

## Optional Features to Add to the Bot

Here are features you could add in the future. Let me know which ones you want and I'll code them.

| Feature | What it does |
|---|---|
| **VPS Renewal** | Allow users to extend their VPS expiry using MoonCoins or payment |
| **Auto Coin Events** | Schedule timed events that reward extra coins to participants |
| **Leaderboard** | `/leaderboard` — top coin holders, top inviters, top VPS users |
| **Daily Reward** | `/daily` — claim a daily coin bonus (cooldown resets every 24h) |
| **VPS Monitoring Alerts** | DM users when their VPS goes above CPU/RAM threshold |
| **Invite Leaderboard** | Track top inviters in the server for rewards |
| **VPS Templates** | Pre-configured server templates (Minecraft, CS2, FiveM, etc.) |
| **Auto DNS** | Automatically assign subdomains via Cloudflare when a VPS is created |
| **VPS Logs Channel** | Mirror all VPS events (created, deleted, suspended) to a log channel |
| **Coin Shop** | `/shop` — spend coins on VPS upgrades, extensions, or custom roles |
| **VPS Stats Graph** | Generate a PNG graph of CPU/RAM usage over time |
| **Multi-language** | Translate bot messages to Filipino, Spanish, etc. |
| **Admin Dashboard** | Web-based dashboard showing all VPS, nodes, and user stats |
| **VPS Expiry Calendar** | Monthly calendar embed showing all upcoming VPS expirations |
| **Referral Codes** | Unique invite codes that give both the referrer and new user bonus coins |
