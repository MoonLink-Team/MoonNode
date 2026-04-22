# 🌙 MoonNodes VPS Manager

A fully-featured Discord bot for managing LXD/Incus VPS containers — with slash commands, multi-node support, a MoonCoin economy, marketplace, staff roles, and more.

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Install the Bot](#2-install-the-bot)
3. [Set Up LXD (single node)](#3-set-up-lxd-single-node)
4. [Set Up Incus (single node)](#4-set-up-incus-single-node)
5. [Multi-Node Setup (LXD)](#5-multi-node-setup-lxd)
6. [Multi-Node Setup (Incus)](#6-multi-node-setup-incus)
7. [Configure the Bot (.env)](#7-configure-the-bot-env)
8. [Run the Bot](#8-run-the-bot)
9. [First Steps in Discord](#9-first-steps-in-discord)
10. [All Commands](#10-all-commands)
11. [Extensions Reference](#11-extensions-reference)
12. [Premium Tiers](#12-premium-tiers)
13. [MoonCoin Economy](#13-mooncoin-economy)
14. [Troubleshooting](#14-troubleshooting)

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

The bot user must be in the `lxd` or `incus-admin` group (see sections 3–4).

---

## 2. Install the Bot

```bash
# 1. Clone or upload the bot files
git clone https://github.com/MoonLink-Team/MoonNode
cd MoonNode
apt install zip
unzip dev.zip

# 2. Install Python dependencies
pip install -r requirements.txt

# 3. Copy and fill in the config
cp .env.example .env
nano .env          # Fill in DISCORD_TOKEN, BOT_OWNER_ID, GUILD_ID at minimum
```

**requirements.txt** installs:
- `discord.py` — Discord API wrapper
- `python-dotenv` — loads `.env` file
- `aiohttp` — async HTTP (used by discord.py)

---

## 3. Set Up LXD (single node)

### Install LXD

```bash
sudo snap install lxd --channel=latest/stable
sudo lxd init --auto
```

> If `lxd init --auto` gives errors, run `sudo lxd init` interactively instead.

### Add the bot user to the lxd group

```bash
sudo usermod -aG lxd $USER
newgrp lxd          # Apply group without logging out
lxc list            # Should work without sudo
```

### Create a storage pool (if not done during init)

```bash
lxc storage create default dir       # Cheapest — directory-based
# OR
lxc storage create default zfs       # Better — needs ZFS installed
# OR
lxc storage create default btrfs     # Good alternative
```

### Set the pool in .env

```env
DEFAULT_STORAGE_POOL=default
```

### Enable privileged containers (needed for VPS functionality)

```bash
lxc profile set default security.privileged true
lxc profile set default security.nesting true
```

### Test

```bash
lxc launch ubuntu:22.04 test-container
lxc list
lxc delete test-container --force
```

---

## 4. Set Up Incus (single node)

Incus is the community fork of LXD. Use it on Ubuntu 24.04+ or if you prefer the newer project.

### Install Incus

```bash
# Ubuntu 22.04 / 24.04
sudo apt-get install -y curl
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sudo sh
sudo incus admin init --auto
```

### Add the bot user to the incus-admin group

```bash
sudo usermod -aG incus-admin $USER
newgrp incus-admin
incus list          # Should work without sudo
```

### Create a storage pool

```bash
incus storage create default dir
# OR for ZFS
incus storage create default zfs
```

### Set .env

```env
DEFAULT_STORAGE_POOL=default
```

### Enable privileged containers

```bash
incus profile set default security.privileged true
incus profile set default security.nesting true
```

### Test

```bash
incus launch images:ubuntu/22.04 test-container
incus list
incus delete test-container --force
```

> **Note:** The bot auto-detects whether `incus` or `lxc` is installed. If both are present, `incus` takes priority. Set `VPS_PREFIX` in `.env` to keep container names consistent.

---

## 5. Multi-Node Setup (LXD)

Multi-node lets you run VPS containers across **multiple physical servers**. The bot connects to remote LXD nodes over the LXD API.

### On each remote node

```bash
# Install LXD
sudo snap install lxd --channel=latest/stable
sudo lxd init --auto

# Enable remote access (TCP listener)
lxc config set core.https_address [::]   # Listen on all interfaces, port 8443
lxc config set core.trust_password "YourSecurePassword"
```

### From the main server (where the bot runs)

```bash
# Add the remote node (you'll be prompted for the trust password)
lxc remote add node2 https://192.168.1.50:8443 --accept-certificate
# Repeat for each additional node: node3, node4, etc.

# Test
lxc list node2:
```

### Register nodes in Discord

Once the bot is running:

```
/node pick_one:Create
```

Fill in the node host, port, and name. The bot stores nodes in the database and uses them for `/create` deployments.

### Enable Multi-Node in Discord

```
/system action:Extension Toggle  extension:Multi-Node Support  ext_state:Enable
```

### How the bot chooses a node

When creating a VPS, the bot calls `get_best_node()` which picks the node with the most free RAM. You can also manually select a node in the `/create` dropdown.

---

## 6. Multi-Node Setup (Incus)

### On each remote node

```bash
sudo apt-get install incus
sudo incus admin init --auto

# Enable remote API
sudo incus config set core.https_address [::]
sudo incus config trust add-certificate /path/to/client.crt
# OR use a token-based approach:
sudo incus config trust add --name bot-server
# This prints a token — use it on the main server
```

### From the main server

```bash
# Using token (recommended)
incus remote add node2 https://192.168.1.50:8443
# Paste the token when prompted

# OR certificate method
incus remote add node2 https://192.168.1.50:8443 --accept-certificate
```

### Verify

```bash
incus list node2:
incus launch images:ubuntu/22.04 node2:test-vps
incus list node2:
incus delete node2:test-vps --force
```

### Register in Discord

```
/node pick_one:Create  host:192.168.1.50  port:8443  name:Node 2
```

Then enable Multi-Node:

```
/system action:Extension Toggle  extension:Multi-Node Support  ext_state:Enable
```

---

## 7. Configure the Bot (.env)

Copy `.env.example` to `.env` and fill in:

```env
# ── REQUIRED ───────────────────────────────────────────────────
DISCORD_TOKEN=your_bot_token_here
BOT_OWNER_ID=123456789012345678      # Your Discord user ID

# ── SERVER ────────────────────────────────────────────────────
GUILD_ID=987654321098765432          # Your server ID (for instant command sync)
                                     # Set to 0 for global sync (takes ~1 hour)

# ── CONTAINER SETTINGS ────────────────────────────────────────
VPS_PREFIX=vps                       # Containers will be named: vps-username-1
DEFAULT_STORAGE_POOL=default         # Name from: lxc storage list

# ── RESOURCE CAPS ─────────────────────────────────────────────
MAX_RAM_GB=16
MAX_CPU_CORES=8

# ── COIN SYSTEM ───────────────────────────────────────────────
MAIN_CHAT_ID=0                       # Channel ID to earn coins by chatting
                                     # Set to 0 to disable

# ── EXTENSIONS ────────────────────────────────────────────────
ACCOUNT=True
MARKETPLACE=True
ISTS=True                            # Enable ticket system

# All others default to False — enable via /system in Discord
```

### Get your Discord IDs

- **Bot Token:** [discord.com/developers](https://discord.com/developers) → Your App → Bot → Reset Token
- **User ID:** Discord → Settings → Advanced → Enable Developer Mode → right-click your name → Copy User ID
- **Server ID:** Right-click your server name → Copy Server ID
- **Channel ID:** Right-click a channel → Copy Channel ID

---

## 8. Run the Bot

```bash
# Direct run
python3 bot.py

# With screen (keeps running after you disconnect)
screen -S moonbot
python3 bot.py
# Press Ctrl+A then D to detach

# With systemd (recommended for production)
sudo nano /etc/systemd/system/moonbot.service
```

**systemd service file:**

```ini
[Unit]
Description=MoonNodes VPS Bot
After=network.target

[Service]
Type=simple
User=youruser
WorkingDirectory=/home/youruser/moonnodes
ExecStart=/usr/bin/python3 bot.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable moonbot
sudo systemctl start moonbot
sudo systemctl status moonbot

# View logs
journalctl -u moonbot -f
```

---

## 9. First Steps in Discord

After the bot starts, you should see `Synced N command(s)` in the logs.

**Step 1 — Set up channels:**
```
/set setting:Log Channel         value:#bot-logs
/set setting:Ticket Channel      value:#support
/set setting:Payment Channel     value:#payments
```

**Step 2 — Add staff (optional):**
```
/role action:Add  user:@YourAdmin  permission:Admin
```

**Step 3 — Create VPS plans:**
```
/plans plan_type:Free  action:Add  plan_name:basic  ram_gb:1  cpu_cores:1  disk_gb:10  price:invite 3
/plans plan_type:Premium  action:Add  plan_name:starter  ram_gb:2  cpu_cores:2  disk_gb:20  price:₱150
```

**Step 4 — Add payment methods:**
```
/payment action:Add/Edit  payment_name:GCash  image:(attach QR code)
```

**Step 5 — Enable extensions you want:**
```
/system action:Extension Toggle  extension:Ticket System  ext_state:Enable
/system action:Extension Toggle  extension:Account & MoonCoin  ext_state:Enable
/system action:Extension Toggle  extension:Marketplace  ext_state:Enable
```

**Step 6 — Test a VPS creation:**
```
/create user:@SomeUser  ram_gb:1  cpu_cores:1  disk_gb:10
```

---

## 10. All Commands

### 👤 User Commands

| Command | Description |
|---|---|
| `/help` | Show available commands based on your role |
| `/list` | View all your VPS instances and their status |
| `/manage` | Full VPS control panel — start, stop, SSH, reinstall, console, backups |
| `/status` | Live host CPU, RAM, and disk stats |
| `/plans plan_type:Free\|Premium` | Browse available VPS plans |
| `/claim` | Claim a Free VPS (invites or coins) or submit a Premium payment |
| `/account` | Balance, coins, VPS sharing, backups, and transfers |
| `/account options:Balance` | View your MoonCoin balance and stats |
| `/account options:VPS user_action:Share` | Grant or revoke VPS access to another user |
| `/account options:VPS user_action:Backup` | Create, list, restore, or delete VPS snapshots |
| `/account options:Transfer` | Send MoonCoins to another user |
| `/upgrade` | Extend your VPS plan / expiry |
| `/backup` | Quick access to VPS backup actions |
| `/support` | Open a private support ticket with staff |
| `/marketplace` | Buy, sell, and browse VPS listings |
| `/affiliate` | View referral credits and stats |
| `/share` | Quick share/unshare a VPS with another user |
| `/transfer` | Transfer a VPS to another user |

### 🛡️ Admin Commands

| Command | Description |
|---|---|
| `/create` | Deploy a VPS for any user — pick node and OS |
| `/delete` | Permanently remove a user's VPS |
| `/vps action:Suspend\|Unsuspend` | Suspend or unsuspend a VPS by container name |
| `/set` | Configure log/ticket/payment channels, thresholds |
| `/payment` | Manage payment method images shown in `/plans` |
| `/giveaway action:Create` | Start a VPS giveaway with timer and auto-deploy |
| `/node` | Manage the LXD/Incus node cluster |
| `/role` | Assign, remove, promote, or demote staff roles |
| `/account admin_action:Coin` | Add, remove, transfer, or reset user MoonCoins |
| `/account admin_action:VPS un_suspend_action:Suspend\|Unsuspend` | Suspend/unsuspend a user's specific VPS |
| `/account admin_action:VPS expire_action:Add\|Edit\|Remove\|View\|Clear` | Manage VPS expiry dates |

### 🔨 Mod Commands

| Command | Description |
|---|---|
| `/mod action:User Info` | Full user profile — VPS count, coins, join date |
| `/mod action:VPS Info` | Detailed container info by name |
| `/mod action:List VPS` | All VPS owned by a specific user |
| `/mod action:Search` | Find VPS by partial container name |
| `/mod action:Server Stats` | Total VPS, users, running, suspended |
| `/mod action:Open Tickets` | List all open support threads |
| `/mod action:Suspension Logs` | Recent VPS suspension history |
| `/mod action:DM User` | Send a staff message to any user |
| `/mod action:Warn User` | Issue a formal warning (saved to DB) |
| `/mod action:Warn Log` | View a user's warning history |

### 🌟 Staff Commands

| Command | Description |
|---|---|
| `/staff action:My Tickets` | Support threads you're assigned to |
| `/staff action:All Tickets` | All open support threads |
| `/staff action:Close Ticket` | Archive and lock a support thread |
| `/staff action:Claim Ticket` | Assign a ticket to yourself |
| `/staff action:Add Note` | Add a private staff note to a user |
| `/staff action:View Notes` | Read a user's staff notes |
| `/staff action:User Lookup` | Quick profile summary |
| `/staff action:Warn User` | Issue a warning (Mod+) |
| `/staff action:Server Overview` | Global stats (Mod+) |
| `/staff action:Staff Announce` | Broadcast DM to all staff (Mod+) |
| `/staff action:Lock\|Unlock Channel` | Lock/unlock a channel (Admin+) |
| `/staff action:Purge` | Delete recent messages (Admin+) |

### 👑 Owner Commands

| Command | Description |
|---|---|
| `/system action:Extension Toggle` | Enable or disable any extension |
| `/system action:View Extensions` | See current on/off state of all extensions |
| `/system action:Maintenance Mode` | Toggle bot maintenance mode |
| `/system action:Clean DB` | Remove stale/empty VPS entries |
| `/system action:A2FA Management` | Set or remove your admin PIN |
| `/system action:MoonCoin Earning` | Add/edit/remove coin earning rules |
| `/system action:List Containers` | List all containers on a node |
| `/system action:Premium` | Assign or remove premium tiers to users |

### 🔧 Dev Commands

| Command | Description |
|---|---|
| `/dev action:Config Dump` | Show all runtime config values |
| `/dev action:DB Stats` | Row counts for all database tables |
| `/dev action:Runtime Info` | Container runtime binary and version |
| `/dev action:Test LXC` | Run `lxc list` and show output |
| `/dev action:Host Stats` | Live CPU, RAM, Disk of the host machine |
| `/dev action:Error Log` | Show last 20 lines of bot.log |
| `/dev action:Reload Cog` | Hot-reload a command module (no restart) |
| `/dev action:Test DM` | Send a test DM to yourself |
| `/dev action:Extension State` | Show all extension flags |
| `/dev action:Purge Cache` | Reload vps_data and admin_data from disk |

---

## 11. Extensions Reference

Enable/disable any extension at runtime via:
```
/system action:Extension Toggle  extension:<name>  ext_state:Enable|Disable
```

| Extension | Description | Config Key |
|---|---|---|
| **Multi-Node Support** | LXD/Incus cluster — deploy across multiple servers | `MULTI_NODE` |
| **Scheduled Backups** | Auto-snapshot all running VPS on a schedule | `SOSB` |
| **Ticket System** | `/support` creates private Discord threads | `ISTS` |
| **Auto Expire & Suspend** | Suspend VPS automatically when expiry date passes | `AES` |
| **Renewal Reminders** | DM users at 7, 3, and 1 day before expiry | `RR` |
| **Admin 2FA** | Require a 4-digit PIN for sensitive admin actions | `A2FA` |
| **Multi-Server Tickets** | Separate ticket channels per Discord server | `MSTC` |
| **Backup Limits** | Cap the number of snapshots per VPS | `UBL_ENABLED` |
| **Marketplace** | Buy/sell VPS for coins or real money | `MARKETPLACE` |
| **Account & MoonCoin** | `/account` command and full coin economy | `ACCOUNT` |
| **VPS Renewal** | Users can extend expiry with `/upgrade` | `VPS_RENEWAL` |
| **VPS Monitoring Alerts** | DM users when CPU/RAM exceeds threshold | `VPS_MONITORING` |
| **VPS Templates** | Minecraft, CS2, Nginx etc. in OS selector | `VPS_TEMPLATES` |
| **Multi-language** | i18n support (stub — needs translation files) | `MULTILANG` |

---

## 12. Premium Tiers

Assign premium tiers to users via:
```
/system action:Premium  premium_action:Set  premium_user:@User  premium_tier:pro
```

| Tier | Icon | Key |
|---|---|---|
| Basic | ⭐ | `basic` |
| Standard | 🌟 | `standard` |
| Pro | 💎 | `pro` |
| Elite | 🔱 | `elite` |

Premium users receive a tier badge in `/mod user_info` and `/account`, and a welcome DM when their tier is assigned.

---

## 13. MoonCoin Economy

MoonCoins are the in-bot currency used to claim free VPS plans and buy from the marketplace.

### Default earning rules

| Action | Reward | Cooldown |
|---|---|---|
| Invite someone | +5 coins | None |
| Chat in main channel | +1 coin | 5 minutes |
| Enter a giveaway | +3 coins | Per giveaway |
| Join an event | +7 coins | Per event |

### Multipliers

- VIP Discord role → **+1%**
- Server Booster → **+2%**
- Multipliers stack

### Admin coin management

```
/account admin_action:Coin  coin_action:Add      target_user:@User  transfer_amount:100
/account admin_action:Coin  coin_action:Remove   target_user:@User  transfer_amount:50
/account admin_action:Coin  coin_action:Reset    target_user:@User
/account admin_action:Coin  coin_action:View     target_user:@User
/account admin_action:Coin  coin_action:Transfer target_user:@Sender  transfer_to:@Receiver  transfer_amount:25
```

### Add custom earning rules

```
/system action:MoonCoin Earning  mooncoin_action:Add  earning_name:daily  earning_amount:10  earning_cooldown:86400
```

---

## 14. Troubleshooting

### "Synced 0 command(s)"

The bot connected but synced no commands. This was a bug in v1 — v2 fixes it. Check:
```
# Ensure GUILD_ID is correct in .env
# Restart the bot after changing GUILD_ID
# Check bot.log for "Failed to load" errors
```

### "lxc: command not found" or "incus: command not found"

```bash
# LXD: add to snap bin path
export PATH=$PATH:/snap/bin
echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc

# Incus: ensure it's in PATH
which incus
```

### Bot can't create containers (permission denied)

```bash
# LXD
sudo usermod -aG lxd $USER
# Log out and back in, or:
newgrp lxd

# Incus
sudo usermod -aG incus-admin $USER
newgrp incus-admin
```

### Containers stuck in "Starting"

```bash
# Check LXD/Incus daemon
sudo systemctl status snap.lxd.daemon
sudo systemctl status incus

# Check logs
lxc info --show-log <container-name>
# OR
incus info --show-log <container-name>
```

### SSH (tmate) not working

The bot installs `tmate` inside the container and requires internet access from within the container. Check:

```bash
# From inside the container
lxc exec <name> -- ping -c 3 8.8.8.8

# If no network, check the profile
lxc profile show default | grep -A5 "network"
```

If there's no `eth0` device in the profile:
```bash
lxc network create lxdbr0
lxc profile device add default eth0 nic nictype=bridged parent=lxdbr0
```

For Incus:
```bash
incus network create incusbr0
incus profile device add default eth0 nic nictype=bridged parent=incusbr0
```

### Multi-node: "connection refused" to remote node

```bash
# On the remote node, verify the listener is active
lxc config get core.https_address
# Should return [::]:8443

# Check firewall
sudo ufw allow 8443/tcp

# Test from the main server
curl -k https://REMOTE_IP:8443/1.0
```

### Database errors or corruption

```bash
# Check bot.log
tail -50 bot.log | grep -i "error\|warn\|corrupt"

# Backup the DB
cp moonnodes.db moonnodes.db.bak

# Rebuild (loses settings — keep vps_data.json)
rm moonnodes.db
python3 -c "from core.database import init_db; init_db()"
```

### bot.log is getting too large

```bash
# Add logrotate
sudo nano /etc/logrotate.d/moonbot
```

```
/home/youruser/moonnodes/bot.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
}
```

---

## File Structure

```
moonnodes/
├── bot.py                     # Entry point — loads cogs, background tasks
├── commands.py                # Consolidated: Role, System, Admin, Deploy, Mod, Dev cogs
├── commands/
│   ├── Admin/
│   │   ├── set_cmd.py         # /set /payment
│   │   └── vps.py             # /vps (admin suspend/unsuspend)
│   ├── Dev/
│   │   └── dev_tools.py       # /dev diagnostics
│   ├── Mod/
│   │   └── mod_tools.py       # /mod moderation tools
│   ├── Owner/
│   │   ├── admin.py           # /role staff management
│   │   ├── create.py          # /create VPS deployment
│   │   ├── delete.py          # /delete VPS removal
│   │   ├── giveaway.py        # /giveaway
│   │   ├── nodes.py           # /node cluster management
│   │   └── system.py          # /system extensions, maintenance
│   ├── Staff/
│   │   └── staff_tools.py     # /staff ticket & moderation tools
│   └── User/
│       ├── account.py         # /account coins, VPS, sharing, expiry
│       ├── affiliate.py       # /affiliate referral credits
│       ├── backup.py          # /backup quick access
│       ├── claim.py           # /claim free/premium VPS
│       ├── help.py            # /help role-based guide
│       ├── list_cmd.py        # /list VPS instances
│       ├── manage.py          # /manage VPS control panel
│       ├── marketplace.py     # /marketplace buy/sell
│       ├── plans.py           # /plans browse and manage
│       ├── share.py           # /share quick share
│       ├── status.py          # /status host stats
│       ├── support.py         # /support open ticket
│       ├── transfer.py        # /transfer VPS to user
│       └── upgrade.py         # /upgrade plan extension
├── core/
│   ├── config.py              # Loads .env, all runtime flags
│   ├── database.py            # SQLite helpers, async-safe lock, connection pool
│   ├── ext.py                 # Extension state checkers
│   ├── lxc.py                 # LXC/Incus command wrappers, deploy logic
│   ├── lxd.py                 # LXD API helpers (multi-node)
│   ├── incus.py               # Incus API helpers (multi-node)
│   ├── monitoring.py          # Host CPU/RAM/Disk monitoring thread
│   ├── cloudflare.py          # Cloudflare DNS subdomain automation
│   └── theme.py               # Embed colors, progress bars, shared builders
├── feature/
│   ├── codes/                 # Redemption codes system
│   └── plans/                 # Plans data and manager
├── ui/
│   ├── confirm_view.py        # Confirm/cancel dialog
│   ├── manage_view.py         # VPS control panel buttons
│   ├── node_select.py         # Node picker dropdown
│   ├── os_select.py           # OS picker dropdown
│   ├── plans_view.py          # Plans and giveaway entry buttons
│   └── software_view.py       # Software installer picker
├── vps_data.json              # In-memory VPS data (auto-created)
├── admin_data.json            # Staff role data (auto-created)
├── moonnodes.db               # SQLite database (auto-created)
├── bot.log                    # Log file (auto-created)
├── .env                       # Your config (never commit this)
└── .env.example               # Template — copy to .env
```

---

*MoonNodes VPS Manager — built with discord.py and LXD/Incus*
