# 🌙 MoonNodes VPS Manager

A full-featured Discord bot for managing **LXD / Incus** containers — slash commands, multi-node clusters, MoonCoin economy, marketplace, dark market, staff hierarchy, automated backups, and more. All bot data is stored in a single **system.db** SQLite file.

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Install the Bot](#2-install-the-bot)
3. [Set Up LXD — Single Node](#3-set-up-lxd--single-node)
4. [Set Up Incus — Single Node](#4-set-up-incus--single-node)
5. [Multi-Node Setup — LXD](#5-multi-node-setup--lxd)
6. [Multi-Node Setup — Incus](#6-multi-node-setup--incus)
7. [Configure the Bot (.env)](#7-configure-the-bot-env)
8. [Run the Bot](#8-run-the-bot)
9. [First Steps in Discord](#9-first-steps-in-discord)
10. [All Commands](#10-all-commands)
11. [Extensions Reference](#11-extensions-reference)
12. [MoonCoin Economy](#12-mooncoin-economy)
13. [Premium Tiers](#13-premium-tiers)
14. [system.db — Unified Database](#14-systemdb--unified-database)
15. [File Structure](#15-file-structure)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. Requirements

| Requirement | Version |
|---|---|
| Python | 3.11 or newer |
| discord.py | 2.3+ |
| LXD **or** Incus | LXD 5.x / Incus 0.5+ |
| OS | Ubuntu 22.04 / 24.04 (recommended) |
| RAM | 2 GB minimum on host |
| Storage | 20 GB minimum |
| git | Any version (for auto-update) |

The user running the bot must be in the `lxd` or `incus-admin` group.

---

## 2. Install the Bot

```bash
# 1. Clone the bot
git clone https://github.com/MoonLink-Team/MoonNode
cd MoonNode

# If you received a zip file instead:
apt install zip
unzip dev.zip
cd MoonNode

# 2. Install Python dependencies
pip install -r requirements.txt

# 3. Copy and fill in config
cp .env.example .env
nano .env   # Fill in DISCORD_TOKEN and BOT_OWNER_ID at minimum
```

---

## 3. Set Up LXD — Single Node

```bash
# Install
sudo snap install lxd --channel=latest/stable
sudo lxd init --auto

# Add bot user to lxd group
sudo usermod -aG lxd $USER
newgrp lxd

# Enable privileged containers
lxc profile set default security.privileged true
lxc profile set default security.nesting true
lxc profile set default security.syscalls.intercept.mknod true

# Create storage pool (if not done during init)
lxc storage create default dir

# Set up networking
lxc network create lxdbr0
lxc profile device add default eth0 nic nictype=bridged parent=lxdbr0

# Test
lxc launch ubuntu:22.04 test-container
lxc list
lxc delete test-container --force
```

---

## 4. Set Up Incus — Single Node

```bash
# Install
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sudo sh
sudo incus admin init --auto

# Add bot user to incus-admin group
sudo usermod -aG incus-admin $USER
newgrp incus-admin

# Enable privileged containers
incus profile set default security.privileged true
incus profile set default security.nesting true
incus profile set default security.syscalls.intercept.mknod true

# Create storage pool
incus storage create default dir
# OR for ZFS (recommended):
incus storage create default zfs

# Set up networking
incus network create incusbr0
incus profile device add default eth0 nic nictype=bridged parent=incusbr0

# Test
incus launch images:ubuntu/22.04 test-container
incus list
incus delete test-container --force
```

> **Root device note:** The bot always passes `--device root,type=disk,path=/,pool=<pool>,size=<N>GB` during `incus init` to prevent the *"No root device could be found"* error.

---

## 5. Multi-Node Setup — LXD

LXD multi-node uses the built-in remote API (TCP port 8443).

### On EACH remote node

```bash
sudo snap install lxd --channel=latest/stable
sudo lxd init --auto

# Enable remote TCP API
lxc config set core.https_address [::]:8443
lxc config set core.trust_password "YourSecurePassword"

# Firewall
sudo ufw allow 8443/tcp

# Storage + profile
lxc storage create default dir
lxc profile set default security.privileged true
lxc profile set default security.nesting true
```

### From the MAIN server

```bash
# Add the remote node
lxc remote add node2 https://192.168.1.50:8443 --accept-certificate
# Enter the trust password when prompted

# Verify
lxc list node2:

# Test launch
lxc launch ubuntu:22.04 node2:test-vps
lxc delete node2:test-vps --force
```

### Register in Discord

```
/node pick_one:Create  name:Node 2  host:192.168.1.50  port:8443  runtime:lxc
/system action:Extension Toggle  extension:Multi-Node Support  ext_state:Enable
```

---

## 6. Multi-Node Setup — Incus

Incus multi-node uses its built-in HTTPS API with token-based auth — **no IP passthrough needed in the bot**, the remote is identified by its registered name.

### On EACH remote node

```bash
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sudo sh
sudo incus admin init --auto

# Enable remote API
sudo incus config set core.https_address [::]:8443

# Generate a trust token for the main server
sudo incus config trust add --name main-bot
# → COPY the printed token

sudo ufw allow 8443/tcp

# Storage + profile
incus storage create default dir
incus profile set default security.privileged true
incus profile set default security.nesting true
```

### From the MAIN server

```bash
# Add remote using token (recommended)
incus remote add node2 https://REMOTE_IP:8443
# Paste the token when prompted

# OR certificate method
incus remote add node2 https://REMOTE_IP:8443 --accept-certificate

# Verify
incus list node2:

# Test
incus launch images:ubuntu/22.04 node2:test-vps
incus delete node2:test-vps --force
```

### Register in Discord

```
/node pick_one:Create  name:Node 2  host:REMOTE_IP  port:8443  runtime:incus
/system action:Extension Toggle  extension:Multi-Node Support  ext_state:Enable
```

### Troubleshoot connection

```bash
# Verify listener on remote node
incus config get core.https_address   # Expected: [::]:8443

# Test from main server
curl -k https://REMOTE_IP:8443/1.0
# Expected: {"type":"sync","status":"Success",...}
```

---

## 7. Configure the Bot (.env)

```env
# ── REQUIRED ──────────────────────────────────────────────────────────────────
DISCORD_TOKEN=your_bot_token_here
BOT_OWNER_ID=123456789012345678

# ── SERVER ────────────────────────────────────────────────────────────────────
GUILD_ID=987654321098765432            # Your server ID

# ── CONTAINER ─────────────────────────────────────────────────────────────────
VPS_PREFIX=vps                         # Container names: vps-username-1
DEFAULT_STORAGE_POOL=default           # Name from: lxc storage list

# ── RESOURCE CAPS ─────────────────────────────────────────────────────────────
MAX_RAM_GB=16
MAX_CPU_CORES=8

# ── COIN SYSTEM ───────────────────────────────────────────────────────────────
MAIN_CHAT_ID=0                         # Channel ID to earn coins by chatting

# ── DISCORD ROLE IDs (0 = not configured) ────────────────────────────────────
OWNER_ROLE_ID=0          # 👑 Owner — full control
HARD_ADMIN_ROLE_ID=0     # 🔱 Hard Admin — senior admin
ADMIN_ROLE_ID=0          # 🛡️ Admin — standard admin
MANAGER_ROLE_ID=0        # 💼 Manager — team manager
HARD_MOD_ROLE_ID=0       # 🔨 Hard Mod — senior moderator
MOD_ROLE_ID=0            # ⚒️ Mod — standard moderator
HARD_STAFF_ROLE_ID=0     # ⭐ Hard Staff — senior support
STAFF_ROLE_ID=0          # 🌟 Staff — support team
TRIAL_STAFF_ROLE_ID=0    # 🔰 Trial Staff — new support
TRIAL_MOD_ROLE_ID=0      # 🎯 Trial Mod — new moderator
PARTNER_ROLE_ID=0        # 🤝 Partner — community partner
VIP_ROLE_ID=0            # 👑 VIP — coin multiplier +1%

# ── CLOUDFLARE (optional) ─────────────────────────────────────────────────────
CF_API_TOKEN=
CF_ZONE_ID=
CF_DOMAIN=yourdomain.com

# ── EXTENSIONS (can also be toggled via /system in Discord) ──────────────────
ACCOUNT=True
MARKETPLACE=True
ISTS=False              # Ticket System
MULTI_NODE=False
DM_COMMANDS=False       # Allow bot commands in DMs
BLOCK_TERMINAL=False    # Block specific terminal commands
```

---

## 8. Run the Bot

```bash
# Direct (testing)
python3 bot.py

# With screen (persists after SSH disconnect)
screen -S moonbot
python3 bot.py
# Ctrl+A then D to detach
# screen -r moonbot to reattach

# With systemd (recommended)
sudo nano /etc/systemd/system/moonbot.service
```

```ini
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
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable moonbot
sudo systemctl start moonbot
sudo systemctl status moonbot
journalctl -u moonbot -f
```

---

## 9. First Steps in Discord

```
# Set up channels
/set setting:Log Channel         value:#bot-logs
/set setting:Ticket Category     value:<CATEGORY_ID>
/set setting:Payment Channel     value:#payments
/set setting:Update Channel      value:#updates

# Add staff
/role action:Add  user:@YourAdmin  permission:Admin

# Create plans
/plans plan_type:Free     action:Add  plan_name:basic    ram_gb:1  cpu_cores:1  disk_gb:10  price:coin 100
/plans plan_type:Premium  action:Add  plan_name:starter  ram_gb:2  cpu_cores:2  disk_gb:20  price:₱150  validity:30 Days

# Add payment method
/payment action:Add/Edit  payment_name:GCash  image:(attach QR image)

# Enable extensions
/system action:Extension Toggle  extension:Ticket System       ext_state:Enable
/system action:Extension Toggle  extension:Account & MoonCoin  ext_state:Enable
/system action:Extension Toggle  extension:Marketplace         ext_state:Enable

# Test VPS creation
/create  user:@SomeUser  plan_name:basic  plan_type:free
```

---

## 10. All Commands

### 👤 User

| Command | Description |
|---|---|
| `/help` | Role-based command guide (tabs per role) |
| `/list` | Your VPS instances and live status |
| `/manage` | VPS control panel — Start, Stop, Restart, SSH, Console, Reinstall |
| `/status` | Host CPU, RAM, Disk stats |
| `/plans plan_type:` | Browse plans (Free / Premium / MoonCoin / Invite) |
| `/claim` | Claim a VPS with invites, coins, or payment proof |
| `/account` | Balance, VPS sharing, backups |
| `/backup` | Create, list, restore VPS snapshots |
| `/upgrade` | Extend or upgrade your VPS plan |
| `/support` | Open a ticket channel in the ticket category |
| `/marketplace` | Buy, sell, browse VPS listings + Dark Market |

### 🛡️ Admin

| Command | Description |
|---|---|
| `/create` | Deploy VPS for any user |
| `/delete` | Permanently delete a VPS |
| `/manage user_id:` | Admin-view any user's VPS panel |
| `/set` | Configure channels, thresholds, blocked commands |
| `/payment` | Manage payment method images |
| `/giveaway` | Start / cancel / list giveaways |
| `/node` | Manage node cluster (List, Status, Create, Edit, Delete, Migrate) |
| `/role` | Assign / promote / demote / list staff |
| `/account admin_action:` | Manage coins, VPS suspension, expiry |

### 🔨 Mod

| Command | Description |
|---|---|
| `/mod action:User Info` | Full user profile |
| `/mod action:VPS Info` | Container details |
| `/mod action:Warn User` | Issue a warning |
| `/mod action:Suspension Logs` | Recent suspensions |

### 🔱 Hard Mod Only

| Command | Description |
|---|---|
| `/hardmod action:Suspend VPS` | Force-suspend a container |
| `/hardmod action:Unsuspend VPS` | Re-enable a container |
| `/hardmod action:Clear Warnings` | Delete all warnings for a user |
| `/hardmod action:Staff Announce` | DM broadcast to all staff |
| `/hardmod action:Lock Channel` | Lock a channel |
| `/hardmod action:Purge` | Delete messages |

### ⭐ Hard Staff Only

| Command | Description |
|---|---|
| `/hardstaff action:User Info` | Full profile + VPS + warns |
| `/hardstaff action:VPS Info` | Container detail lookup |
| `/hardstaff action:Warn User` | Issue a warning |
| `/hardstaff action:DM User` | Send staff message to user |
| `/hardstaff action:Open Tickets` | All open ticket channels |
| `/hardstaff action:Suspension Logs` | Recent suspension history |

### 👑 Owner

| Command | Description |
|---|---|
| `/system action:Extension Toggle` | Enable/disable extensions |
| `/system action:Maintenance Mode` | Toggle maintenance + post to update channel |
| `/system action:Update Bot` | Pull latest code from GitHub |
| `/system action:MoonCoin Earning` | Manage coin rules |
| `/system action:Clean DB` | Remove stale VPS entries |
| `/system action:A2FA Management` | Set admin PIN |

---

## 11. Extensions Reference

Toggle any extension at runtime — no restart needed:

```
/system action:Extension Toggle  extension:<name>  ext_state:Enable|Disable
```

| Extension | What it does |
|---|---|
| Multi-Node Support | Deploy across LXD/Incus cluster |
| Scheduled Backups | Auto-snapshot on a schedule |
| Ticket System | `/support` creates channels in a category |
| Auto Expire & Suspend | Auto-suspend expired VPS |
| Renewal Reminders | DM users before expiry |
| Admin 2FA | PIN required for sensitive admin actions |
| Backup Limits | Cap max snapshots per VPS |
| Marketplace | Buy/sell VPS with coins or money |
| Account & MoonCoin | `/account` + full coin economy |
| VPS Renewal | Users extend expiry with `/upgrade` |
| VPS Monitoring Alerts | DM when CPU/RAM exceeds threshold |
| **User Can DM Commands** | Allow bot commands in DMs |
| **Block Terminal Commands** | Prevent specific commands in VPS terminals |

---

## 12. MoonCoin Economy

| Action | Coins | Cooldown |
|---|---|---|
| Invite someone (valid — stayed 7 days) | +5 | — |
| Chat in main channel | +1 | 5 minutes |
| Enter a giveaway | +3 | per giveaway |
| Join an event | +7 | per event |

**Multipliers:** VIP role +1% · Server Booster +2% (stack)

**Anti-fake invite rule:** If an invited user leaves within 7 days, the invite is invalidated and 5 coins are deducted from the inviter.

**Manage rules:**
```
/system action:MoonCoin Earning  mooncoin_action:Add
  earning_name:daily  earning_amount:10  earning_cooldown:86400
```

---

## 13. Premium Tiers

```
/system action:Premium  premium_action:Set  premium_user:@User  premium_tier:pro
```

| Tier | Key |
|---|---|
| Basic ⭐ | `basic` |
| Standard 🌟 | `standard` |
| Pro 💎 | `pro` |
| Elite 🔱 | `elite` |

---

## 14. system.db — Unified Database

All bot data is stored in a **single SQLite file**: `system.db`.

| Table | Contents |
|---|---|
| `vps_entries` | All VPS records (replaces vps_data.json) |
| `users` | Coins, invite counts, join dates |
| `admins` | Staff roles |
| `settings` | Channel IDs, thresholds, config |
| `payments` | Payment method images |
| `marketplace` | User VPS listings |
| `dark_market` | Admin-created VPS listings |
| `warnings` | User warnings |
| `invite_tracking` | Invite validity (7-day anti-fake) |
| `claim_requests` | Premium claim submissions |
| `mooncoin_rules` | Custom earning rules |
| `blocked_commands` | Terminal commands blocked by admin |

**Migration:** If `vps_data.json` exists, it is automatically imported into `vps_entries` on first run, then renamed to `vps_data.json.migrated`.

**Backup:**
```bash
cp system.db system.db.bak
```

---

## 15. File Structure

```
MoonNode/
├── bot.py                          # Entry point — cogs, events, on_member_join/remove
├── system.db                       # Unified database (auto-created)
├── commands/
│   ├── Admin/
│   │   └── set_cmd.py              # /set  /payment
│   ├── Mod/
│   │   ├── mod_tools.py            # /mod
│   │   └── hardmod.py              # /hardmod  (Hard Mod+)
│   ├── Owner/
│   │   ├── admin.py                # /role
│   │   ├── create.py               # /create
│   │   ├── delete.py               # /delete
│   │   ├── giveaway.py             # /giveaway
│   │   ├── nodes.py                # /node
│   │   └── system.py               # /system  (Owner)
│   ├── Staff/
│   │   ├── staff_tools.py          # /staff
│   │   └── hardstaff.py            # /hardstaff  (Hard Staff+)
│   └── User/
│       ├── account.py              # /account
│       ├── backup.py               # /backup
│       ├── claim.py                # /claim
│       ├── help.py                 # /help
│       ├── list_cmd.py             # /list
│       ├── manage.py               # /manage
│       ├── marketplace.py          # /marketplace + dark market
│       ├── plans.py                # /plans
│       ├── status.py               # /status
│       ├── support.py              # /support  (channel-based tickets)
│       └── upgrade.py              # /upgrade
├── core/
│   ├── config.py                   # .env loader, all flags + role IDs
│   ├── database.py                 # system.db layer, vps_data cache
│   ├── ext.py                      # Extension flag helpers
│   ├── lxc.py                      # Runtime router (Incus or LXC)
│   ├── incus.py                    # Incus backend
│   ├── lxc_backend.py              # LXC/LXD backend
│   ├── lxd.py                      # Shim → lxc_backend (legacy compat)
│   ├── monitoring.py               # Host stats monitoring
│   ├── cloudflare.py               # DNS automation
│   └── theme.py                    # Embeds, colours, shared UI
├── ui/
│   ├── manage_view.py              # VPS control panel buttons
│   ├── node_select.py              # Node picker dropdown
│   ├── os_select.py                # OS picker dropdown
│   ├── plans_view.py               # Plans / giveaway buttons
│   └── confirm_view.py             # Confirm / cancel dialog
├── feature/plans/                  # Plans data + manager
├── .env                            # Your config (never commit)
└── .env.example                    # Template
```

---

## 16. Troubleshooting

### VPS creation — "No root device could be found"

The bot now always passes `--device root,type=disk,path=/,pool=<pool>,size=<N>GB` during `incus init`. If you still see this error, your storage pool name might be wrong. Check:

```bash
incus storage list
```

Set the correct pool in `.env`:
```env
DEFAULT_STORAGE_POOL=default   # or: local, zfs, btrfs — whatever incus storage list shows
```

### Snapshot error — "unknown command for incus snapshot"

Incus ≥0.5 changed the snapshot syntax. The bot auto-detects and falls back. If you still see issues:

```bash
incus --version
# If ≥0.5: new syntax is  incus snapshot create <container> <n>
# If <0.5: old syntax is  incus snapshot <container> <n>
```

### "AttributeError: \__enter__"

`data_lock` must be `asyncio.Lock()`. Never use `with data_lock:` — always `async with data_lock:`. This is fixed in all files.

### "async_save_vps_data is not defined"

Import it from `core.database`:
```python
from core.database import vps_data, async_save_vps_data, data_lock
```

### Bot cannot create containers (permission denied)

```bash
# LXD
sudo usermod -aG lxd $USER && newgrp lxd

# Incus
sudo usermod -aG incus-admin $USER && newgrp incus-admin

# Verify
lxc list    # or: incus list
```

### Synced 0 commands

Check `GUILD_ID` in `.env`. Restart the bot after changing it.

### Multi-node — connection refused

```bash
# Remote node
incus config get core.https_address   # Must be [::]:8443
sudo ufw status                        # 8443 must be open

# From main server
curl -k https://REMOTE_IP:8443/1.0    # Must return JSON
```

### Auto-update fails

The bot folder must be a git repo:
```bash
cd /root/MoonNode
git remote -v    # Should show MoonLink-Team/MoonNode
git status
```

If not a git repo:
```bash
git init
git remote add origin https://github.com/MoonLink-Team/MoonNode
git fetch origin main
git reset --hard origin/main
```
