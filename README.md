# 🌙 MoonNodes VPS Manager

A fully-featured Discord bot for managing **LXD / Incus** VPS containers — slash commands, multi-node clustering, API-key-based node connections, a MoonCoin economy, marketplace, dark market, staff hierarchy, automated backups, giveaways, ticket system, and more.

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Install the Bot](#2-install-the-bot)
3. [Set Up LXD — Single Node](#3-set-up-lxd--single-node)
4. [Set Up Incus — Single Node](#4-set-up-incus--single-node)
5. [Multi-Node — LXD (API Key)](#5-multi-node--lxd-api-key)
6. [Multi-Node — Incus (API Key)](#6-multi-node--incus-api-key)
7. [Configure the Bot (.env)](#7-configure-the-bot-env)
8. [Run the Bot](#8-run-the-bot)
9. [First Steps in Discord](#9-first-steps-in-discord)
10. [All Commands](#10-all-commands)
11. [Node Sections](#11-node-sections)
12. [Extensions Reference](#12-extensions-reference)
13. [Premium Tiers](#13-premium-tiers)
14. [MoonCoin Economy](#14-mooncoin-economy)
15. [Role Hierarchy](#15-role-hierarchy)
16. [File Structure](#16-file-structure)
17. [Troubleshooting](#17-troubleshooting)

---

## 1. Requirements

| Requirement | Version |
|---|---|
| Python | 3.11 or newer |
| discord.py | 2.3+ |
| LXD **or** Incus | LXD 5.x / Incus 0.5+ |
| OS | Ubuntu 22.04 / 24.04 (recommended) |
| RAM | 2 GB minimum on the host |
| Storage | 20 GB minimum |

---

## 2. Install the Bot

```bash
# 1. Clone the repo
git clone https://github.com/MoonLink-Team/MoonNode
cd MoonNode

# Or if you have a zip file:
apt install zip
unzip dev.zip
cd MoonNode

# 2. Install Python dependencies
pip install -r requirements.txt

# 3. Copy the config template
cp .env.example .env
nano .env    # fill in DISCORD_TOKEN and BOT_OWNER_ID at minimum
```

---

## 3. Set Up LXD — Single Node

```bash
# Install LXD
sudo snap install lxd --channel=latest/stable
sudo lxd init --auto

# Add bot user to lxd group
sudo usermod -aG lxd $USER
newgrp lxd
lxc list   # verify it works without sudo

# Enable privileged containers
lxc profile set default security.privileged true
lxc profile set default security.nesting true
lxc profile set default security.syscalls.intercept.mknod true

# Set up networking
lxc network create lxdbr0
lxc profile device add default eth0 nic nictype=bridged parent=lxdbr0

# Test
lxc launch ubuntu:22.04 test-container
lxc list
lxc exec test-container -- ping -c 3 8.8.8.8
lxc delete test-container --force
```

---

## 4. Set Up Incus — Single Node

```bash
# Install Incus
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sudo sh
sudo incus admin init --auto

# Add bot user to incus-admin group
sudo usermod -aG incus-admin $USER
newgrp incus-admin
incus list   # verify

# Enable privileged containers
incus profile set default security.privileged true
incus profile set default security.nesting true
incus profile set default security.syscalls.intercept.mknod true

# Set up networking
incus network create incusbr0
incus profile device add default eth0 nic nictype=bridged parent=incusbr0

# Test
incus launch images:ubuntu/22.04 test-container
incus list
incus exec test-container -- ping -c 3 8.8.8.8
incus delete test-container --force
```

---

## 5. Multi-Node — LXD (API Key)

MoonNodes uses **API key tokens** to connect to remote nodes — no IP address entry needed in Discord. The token contains the address, certificate, and credentials automatically.

### On EACH remote node

```bash
# 1. Install LXD
sudo snap install lxd --channel=latest/stable
sudo lxd init --auto

# 2. Enable the remote API listener
lxc config set core.https_address [::]:8443

# 3. Generate a join token for the bot
lxc config trust add --name moonbot
# → COPY the printed token — this is your API key

# 4. Open the port
sudo ufw allow 8443/tcp

# 5. Configure the profile
lxc profile set default security.privileged true
lxc profile set default security.nesting true
lxc profile set default security.syscalls.intercept.mknod true
lxc network create lxdbr0
lxc profile device add default eth0 nic nictype=bridged parent=lxdbr0
```

### Register in Discord (bot running on main server)

```
/node pick_one:Create  name:node2  runtime:lxc  api_key:<paste token here>  section:All
```

The bot automatically runs `lxc remote add node2 <token>` in the background. No IP, no certificate steps — the token handles everything.

### Verify

```
/node pick_one:Status
```

You should see live RAM/CPU/Disk stats for `node2`.

---

## 6. Multi-Node — Incus (API Key)

### On EACH remote node

```bash
# 1. Install Incus
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sudo sh
sudo incus admin init --auto

# 2. Enable the remote API listener
sudo incus config set core.https_address [::]:8443

# 3. Generate a join token
sudo incus config trust add --name moonbot
# → COPY the printed token — this is your API key

# 4. Open the port
sudo ufw allow 8443/tcp

# 5. Configure the profile
incus profile set default security.privileged true
incus profile set default security.nesting true
incus profile set default security.syscalls.intercept.mknod true
incus network create incusbr0
incus profile device add default eth0 nic nictype=bridged parent=incusbr0
```

### Register in Discord

```
/node pick_one:Create  name:node2  runtime:incus  api_key:<paste token here>  section:All
```

### How the token connection works

```
incus config trust add --name moonbot
# Outputs a token like:
# eyJhZGRyZXNzZXMiOlsiMTkyLjE2OC4xLjUwOjg0NDMiXSwiZmluZ2VycHJpbnQi...

# The bot automatically runs:
incus remote add node2 <token>
# This embeds the address, fingerprint, and credentials — no separate IP needed
```

### Troubleshoot API key connection

```bash
# Verify listener on remote node
incus config get core.https_address
# Expected: [::]:8443

# Test reachability from the main server
curl -k https://REMOTE_IP:8443/1.0
# Expected: JSON with "status":"Success"
```

---

## 7. Configure the Bot (.env)

```env
# ── REQUIRED ──────────────────────────────────────────────────────────────────
DISCORD_TOKEN=your_bot_token_here
BOT_OWNER_ID=123456789012345678        # Your Discord user ID

# ── SERVER ────────────────────────────────────────────────────────────────────
GUILD_ID=987654321098765432            # Your server ID (instant command sync)

# ── CONTAINER ─────────────────────────────────────────────────────────────────
VPS_PREFIX=vps                         # Containers: vps-username-1
DEFAULT_STORAGE_POOL=default           # lxc storage list

# ── RESOURCE LIMITS ───────────────────────────────────────────────────────────
MAX_RAM_GB=16
MAX_CPU_CORES=8
ENFORCE_RESOURCE_LIMITS=True

# ── COIN SYSTEM ───────────────────────────────────────────────────────────────
MAIN_CHAT_ID=0                         # Channel ID to earn coins by chatting

# ── ROLE IDs (Discord Role IDs — 0 = not configured) ─────────────────────────
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
PARTNER_ROLE_ID=0        # 🤝 Partner
VIP_ROLE_ID=0            # 👑 VIP — coin multiplier +1%

# ── CLOUDFLARE (optional) ─────────────────────────────────────────────────────
ASM=False
CF_API_TOKEN=
CF_ZONE_ID=
CF_DOMAIN=yourdomain.com

# ── EXTENSIONS ────────────────────────────────────────────────────────────────
ACCOUNT=True
MARKETPLACE=True
ISTS=False             # Ticket system
```

---

## 8. Run the Bot

```bash
# Direct (for testing)
python3 bot.py
```

### With systemd (recommended for production)

```bash
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
journalctl -u moonbot -f   # follow logs
```

---

## 9. First Steps in Discord

```
/set setting:Log Channel      value:#bot-logs
/set setting:Ticket Channel   value:#support
/set setting:Payment Channel  value:#payments

/role action:Add  user:@YourAdmin  permission:Admin

/plans plan_type:Free    action:Add  plan_name:basic    ram_gb:1  cpu_cores:1  disk_gb:10  price:invite 3
/plans plan_type:Premium action:Add  plan_name:starter  ram_gb:2  cpu_cores:2  disk_gb:20  price:₱150

/system action:Extension Toggle  extension:Ticket System       ext_state:Enable
/system action:Extension Toggle  extension:Account & MoonCoin  ext_state:Enable
/system action:Extension Toggle  extension:Marketplace         ext_state:Enable

/create  user:@TestUser  plan_name:basic  plan_type:Free
```

---

## 10. All Commands

### 👤 User

| Command | Description |
|---|---|
| `/help` | Show commands for your role (interactive, tabbed) |
| `/list` | View all your VPS and their live status |
| `/manage` | VPS control panel — Start, Stop, Restart, SSH, Console, Reinstall |
| `/status` | Live host CPU, RAM, Disk stats |
| `/plans` | Browse Free and Premium VPS plans |
| `/claim` | Claim a Free (invites/coins) or Premium (payment proof) VPS |
| `/account` | View balance, manage VPS sharing and backups |
| `/backup` | Quick VPS snapshot access |
| `/upgrade` | Upgrade or extend your VPS plan |
| `/support` | Open a private support ticket |
| `/marketplace` | Buy, sell VPS — includes 🌑 Dark Market (admin listings) |

### 🛡️ Admin

| Command | Description |
|---|---|
| `/create` | Deploy a VPS for any user — pick plan, node, OS |
| `/delete` | Permanently remove a user's VPS |
| `/manage user_id:` | Admin-view any user's VPS control panel |
| `/list action:All VPS` | View all VPS across every user |
| `/node` | Manage the node cluster — Create, Status, Migrate, Delete |
| `/role` | Manage staff roles |
| `/set` | Configure channels, thresholds, and schedules |
| `/payment` | Manage payment method images |
| `/giveaway` | Start a VPS giveaway — admin picks node, winner picks OS via DM |
| `/account admin_action:*` | Manage coins, suspensions, expiry |
| `/system` | Extensions, maintenance, premium, MoonCoin rules |

### 🔨 Mod

| Command | Description |
|---|---|
| `/mod` | User info, VPS info, warnings, DM, server stats, ticket list |

### 🔱 Hard Mod (Hard Mod+)

| Command | Description |
|---|---|
| `/hardmod action:Suspend VPS` | Force-suspend a VPS |
| `/hardmod action:Unsuspend VPS` | Unsuspend a VPS |
| `/hardmod action:Clear Warnings` | Remove all warnings for a user |
| `/hardmod action:Staff Announce` | DM broadcast to all staff |
| `/hardmod action:Lock Channel` | Lock a channel |
| `/hardmod action:Unlock Channel` | Unlock a channel |
| `/hardmod action:Purge` | Delete up to 100 messages |

### 🌟 Staff

| Command | Description |
|---|---|
| `/staff` | My Tickets, All Tickets, Claim, Close, User Lookup, Notes |

### ⭐ Hard Staff (Hard Staff+)

| Command | Description |
|---|---|
| `/hardstaff action:User Info` | Full user profile with VPS, coins, warns |
| `/hardstaff action:VPS Info` | Detailed container info |
| `/hardstaff action:List VPS` | All VPS owned by a user |
| `/hardstaff action:Server Stats` | Global VPS counts |
| `/hardstaff action:Open Tickets` | All open ticket threads |
| `/hardstaff action:Suspension Logs` | Recent VPS suspension history |
| `/hardstaff action:DM User` | Send a staff message to any user |
| `/hardstaff action:Warn User` | Issue a formal warning |
| `/hardstaff action:Warn Log` | View a user's warning history |

### 💻 Dev

| Command | Description |
|---|---|
| `/dev` | Config dump, DB stats, runtime info, error log, hot-reload cogs |

---

## 11. Node Sections

When registering a node with `/node pick_one:Create`, set its **Section** to control who can deploy there:

| Section | Who can deploy | Use case |
|---|---|---|
| 🌙 MoonCoin & Invite & Giveaway | Free-plan users only | Budget/free tier nodes |
| 💎 Premium Users Only | Premium plan users only | High-performance paid nodes |
| 🌐 All Users | Everyone | General purpose |

The node selector automatically filters nodes based on the plan type being claimed or created. Free-plan users never see Premium-only nodes, and vice versa.

---

## 12. Extensions Reference

```
/system action:Extension Toggle  extension:<name>  ext_state:Enable|Disable
```

| Extension | Description |
|---|---|
| **Multi-Node Support** | Deploy across LXD/Incus cluster via API keys |
| **Scheduled Backups** | Auto-snapshot all running containers |
| **Ticket System** | `/support` creates private Discord threads |
| **Auto Expire & Suspend** | Suspend VPS when expiry passes |
| **Renewal Reminders** | DM users at 7, 3, 1 day before expiry |
| **Admin 2FA** | PIN required for sensitive admin actions |
| **Marketplace** | Buy and sell VPS for coins or money |
| **Dark Market** | Admin-only listings, users buy with coins |
| **Account & MoonCoin** | `/account` and the full coin economy |
| **VPS Renewal** | Users extend expiry with `/upgrade` |
| **VPS Monitoring** | DM alerts when CPU/RAM exceeds threshold |
| **VPS Templates** | Minecraft, CS2, Nginx etc. in OS selector |

---

## 13. Premium Tiers

```
/system action:Premium  premium_action:Set  premium_user:@User  premium_tier:pro
```

| Tier | Key |
|---|---|
| ⭐ Basic | `basic` |
| 🌟 Standard | `standard` |
| 💎 Pro | `pro` |
| 🔱 Elite | `elite` |

---

## 14. MoonCoin Economy

| Action | Coins | Cooldown |
|---|---|---|
| Invite someone (stays 7+ days) | +5 | — |
| Chat in main channel | +1 | 5 minutes |
| Enter a giveaway | +3 | Per giveaway |
| Join an event | +7 | Per event |
| VIP Discord role | +1% multiplier | — |
| Server Boost | +2% multiplier | — |

**Anti-fake invite protection:** If an invited user leaves within **7 days**, their invite is invalidated and the 5 coins are automatically clawed back from the inviter.

### Admin coin management

```
/account admin_action:Coin  coin_action:Add     target_user:@User  transfer_amount:100
/account admin_action:Coin  coin_action:Remove  target_user:@User  transfer_amount:50
/account admin_action:Coin  coin_action:View    target_user:@User
```

### Custom earning rules

```
/system action:MoonCoin Earning  mooncoin_action:Add  earning_name:daily  earning_amount:10
```

---

## 15. Role Hierarchy

Roles are managed via `/role` and stored in `admin_data.json`. The Discord Role IDs in `.env` are optional — they're used for display only; permission checks use the `admin_data.json` store.

```
👑  Owner       — Full control, all commands
🔱  Hard Admin  — All admin + senior mod tools
🛡️  Admin       — VPS create/delete, node management, giveaways
💼  Manager     — Admin tools without node management
🔨  Hard Mod    — Mod + suspend/unsuspend, lock channels, broadcast
⚒️  Mod         — Lookups, warnings, DMs, ticket management
⭐  Hard Staff  — Staff + extended lookups, suspend logs
🌟  Staff       — Ticket claim/close, user notes, user lookup
🔰  Trial Staff — Basic staff tools, ticket viewing
🎯  Trial Mod   — Ticket viewing, user lookup
🤝  Partner     — User-level access
👑  VIP         — +1% MoonCoin multiplier
```

---

## 16. File Structure

```
MoonNode/
├── bot.py                         # Entry point — loads cogs, starts tasks
├── commands/
│   ├── Admin/
│   │   ├── set_cmd.py             # /set  /payment
│   │   └── vps.py                 # /vps admin tools (removed in v2)
│   ├── Dev/
│   │   └── dev_tools.py           # /dev
│   ├── Mod/
│   │   ├── mod_tools.py           # /mod
│   │   └── hardmod.py             # /hardmod (Hard Mod+)
│   ├── Owner/
│   │   ├── admin.py               # /role
│   │   ├── create.py              # /create
│   │   ├── delete.py              # /delete
│   │   ├── giveaway.py            # /giveaway (admin node select + winner DM OS select)
│   │   ├── nodes.py               # /node (API key based, section support)
│   │   └── system.py              # /system
│   ├── Staff/
│   │   ├── staff_tools.py         # /staff
│   │   └── hardstaff.py           # /hardstaff (Hard Staff+)
│   └── User/
│       ├── account.py             # /account
│       ├── backup.py              # /backup
│       ├── claim.py               # /claim (admin Continue/Cancel buttons)
│       ├── help.py                # /help (Staff, Hard Staff, Hard Mod tabs)
│       ├── list_cmd.py            # /list
│       ├── manage.py              # /manage (no Install Software)
│       ├── marketplace.py         # /marketplace + Dark Market
│       ├── plans.py               # /plans
│       ├── status.py              # /status
│       ├── support.py             # /support (Add User button in tickets)
│       └── upgrade.py             # /upgrade
├── core/
│   ├── config.py                  # .env loader — all flags + role IDs
│   ├── database.py                # SQLite + async lock + invite_tracking + dark_market
│   ├── ext.py                     # Extension state checkers
│   ├── incus.py                   # Incus backend (name fix, double-colon fix, cgroup2 fix)
│   ├── lxc.py                     # Router — detects Incus or LXD, loads backend
│   ├── lxc_backend.py             # LXC/LXD backend
│   ├── lxd.py                     # Compatibility shim → lxc_backend
│   ├── monitoring.py              # Host resource monitoring
│   ├── cloudflare.py              # Cloudflare DNS
│   └── theme.py                   # Embed colours, progress bars
├── feature/
│   └── plans/                     # Plans data + manager
├── ui/
│   ├── confirm_view.py            # Confirm/cancel dialog
│   ├── manage_view.py             # VPS control panel (Install Software removed)
│   ├── node_select.py             # Node picker — section filter + rich stats
│   ├── os_select.py               # OS picker → triggers deploy
│   └── plans_view.py             # Plans + giveaway entry buttons
├── vps_data.json                  # VPS records (auto-created)
├── admin_data.json                # Staff roles (auto-created)
├── moonnodes.db                   # SQLite database (auto-created)
├── bot.log                        # Log file (auto-created)
├── .env                           # Your config
└── .env.example                   # Template
```

---

## 17. Troubleshooting

### `AttributeError: __enter__` on `data_lock`

`data_lock` is `asyncio.Lock` — it must be used with `async with`, not `with`. Fixed in `core/lxc.py` and `commands/User/manage.py`.

### `Invalid instance name` from Incus

Incus only allows alphanumeric characters and hyphens in container names. The bot now sanitizes names automatically — underscores, spaces, and special characters are replaced with hyphens.

### Double-colon in Incus command (`node2::vps-name`)

Fixed in `core/incus.py` — the `node` value is stripped of trailing colons before building the prefix.

### `NameError: name 'async_save_vps_data' is not defined`

`delete.py` now correctly imports `async_save_vps_data` from `core.database`.

### `UnboundLocalError: local variable 'create_error_embed'`

`create.py` now imports `create_error_embed` at the top level.

### Bot can't connect to remote node

```bash
# Verify listener on remote node
incus config get core.https_address   # → [::]:8443

# Verify port is open
sudo ufw status
sudo ufw allow 8443/tcp

# Test from main server
curl -k https://REMOTE_IP:8443/1.0
```

### VPS not deleted when member leaves

Requires `Manage Server` permission for the bot to track invites. Make sure the bot has this permission.

### Bot can't create containers (permission denied)

```bash
sudo usermod -aG lxd $USER && newgrp lxd        # LXD
sudo usermod -aG incus-admin $USER && newgrp incus-admin  # Incus
```

### `lxc: command not found`

```bash
export PATH=$PATH:/snap/bin
echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc
source ~/.bashrc
```

---

*MoonNodes VPS Manager — Built with discord.py · LXD · Incus*
