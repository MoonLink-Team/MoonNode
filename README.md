# 🌙 MoonNodes VPS Manager

A fully-featured Discord bot for managing **LXD / Incus** VPS containers — slash commands, multi-node clustering, a MoonCoin economy, marketplace, staff hierarchy, automated backups, and more.

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
12. [Premium Tiers](#12-premium-tiers)
13. [MoonCoin Economy](#13-mooncoin-economy)
14. [File Structure](#14-file-structure)
15. [Troubleshooting](#15-troubleshooting)

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

The user running the bot must be in the `lxd` or `incus-admin` group (see sections 3–4).

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
nano .env   # Fill in DISCORD_TOKEN, BOT_OWNER_ID, GUILD_ID at minimum
```

**requirements.txt installs:**
- `discord.py` — Discord API wrapper  
- `python-dotenv` — loads `.env` file  
- `aiohttp` — async HTTP (used by discord.py)

---

## 3. Set Up LXD — Single Node

### Install LXD

```bash
sudo snap install lxd --channel=latest/stable
sudo lxd init --auto
# If --auto errors, run interactively:
# sudo lxd init
```

### Add the bot user to the lxd group

```bash
sudo usermod -aG lxd $USER
newgrp lxd        # Apply without logging out
lxc list          # Should work without sudo
```

### Create a storage pool (if skipped during init)

```bash
lxc storage create default dir        # Cheapest — directory-based
# OR
lxc storage create default zfs        # Recommended — needs ZFS
# OR
lxc storage create default btrfs      # Good alternative
```

### Set the pool in .env

```env
DEFAULT_STORAGE_POOL=default
```

### Enable privileged containers (required for VPS functionality)

```bash
lxc profile set default security.privileged true
lxc profile set default security.nesting true
lxc profile set default security.syscalls.intercept.mknod true
```

### Set up networking (containers need internet access)

```bash
# Check if a bridge already exists
lxc network list

# If not, create one
lxc network create lxdbr0
lxc profile device add default eth0 nic nictype=bridged parent=lxdbr0
```

### Test everything

```bash
lxc launch ubuntu:22.04 test-container
lxc list
lxc exec test-container -- ping -c 3 8.8.8.8
lxc delete test-container --force
```

---

## 4. Set Up Incus — Single Node

Incus is the community fork of LXD. Use it on Ubuntu 24.04+ or if you prefer the newer project. The bot auto-detects whichever is installed — if both are present, **Incus takes priority**.

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
# OR for ZFS (recommended)
incus storage create default zfs
```

### Enable privileged containers

```bash
incus profile set default security.privileged true
incus profile set default security.nesting true
incus profile set default security.syscalls.intercept.mknod true
```

### Set up networking

```bash
incus network list

# If no bridge exists:
incus network create incusbr0
incus profile device add default eth0 nic nictype=bridged parent=incusbr0
```

### Test everything

```bash
incus launch images:ubuntu/22.04 test-container
incus list
incus exec test-container -- ping -c 3 8.8.8.8
incus delete test-container --force
```

---

## 5. Multi-Node Setup — LXD

Multi-node lets you run VPS containers across **multiple physical servers**. The bot routes creation and management to the correct node automatically.

### On EACH remote node

```bash
# 1. Install LXD
sudo snap install lxd --channel=latest/stable
sudo lxd init --auto

# 2. Enable the remote TCP API
lxc config set core.https_address [::]:8443
lxc config set core.trust_password "YourSecurePassword"

# 3. Open the port
sudo ufw allow 8443/tcp

# 4. Set up storage and profile (same as single-node, section 3)
lxc storage create default dir
lxc profile set default security.privileged true
lxc profile set default security.nesting true
```

### From the MAIN server (where the bot runs)

```bash
# Add each remote node — you'll be prompted for the trust password
lxc remote add node2 https://192.168.1.50:8443 --accept-certificate
lxc remote add node3 https://192.168.1.51:8443 --accept-certificate

# Verify the connection
lxc list node2:
lxc list node3:

# Test launching on a remote node
lxc launch ubuntu:22.04 node2:test-vps
lxc list node2:
lxc delete node2:test-vps --force
```

### Register nodes in Discord

Once the bot is running:

```
/node pick_one:Create  name:Node 2  host:192.168.1.50  port:8443
/node pick_one:Create  name:Node 3  host:192.168.1.51  port:8443
/node pick_one:List
```

### Enable Multi-Node

```
/system action:Extension Toggle  extension:Multi-Node Support  ext_state:Enable
```

### How node selection works

When creating a VPS the bot calls `get_best_node()` which picks the node with the most free RAM. Admins can also manually select a node in the `/create` dropdown. Admins can migrate an existing VPS between nodes at any time:

```
/node pick_one:Migrate  vps_name:vps-user-1  to_node:node3
```

---

## 6. Multi-Node Setup — Incus

### On EACH remote node

```bash
# 1. Install Incus
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sudo sh
sudo incus admin init --auto

# 2. Enable the remote API
sudo incus config set core.https_address [::]:8443

# 3. Generate a join token for the main server
sudo incus config trust add --name main-bot
# → Copy the printed token

# 4. Open the port
sudo ufw allow 8443/tcp

# 5. Set up storage and profile
incus storage create default dir
incus profile set default security.privileged true
incus profile set default security.nesting true
```

### From the MAIN server

```bash
# Method A — token (recommended)
incus remote add node2 https://192.168.1.50:8443
# Paste the token when prompted

# Method B — certificate (if token method fails)
incus remote add node2 https://192.168.1.50:8443 --accept-certificate

# Verify
incus list node2:

# Test
incus launch images:ubuntu/22.04 node2:test-vps
incus list node2:
incus delete node2:test-vps --force
```

### Register and enable in Discord

```
/node pick_one:Create  name:Node 2  host:192.168.1.50  port:8443
/system action:Extension Toggle  extension:Multi-Node Support  ext_state:Enable
```

### Troubleshoot remote connections

```bash
# Verify listener is running on the remote node
incus config get core.https_address
# Expected: [::]:8443

# Test from main server
curl -k https://REMOTE_IP:8443/1.0
# Expected: JSON with {"type":"sync","status":"Success",...}
```

---

## 7. Configure the Bot (.env)

Copy `.env.example` to `.env` and fill in these values:

```env
# ── REQUIRED ──────────────────────────────────────────────────────────────────
DISCORD_TOKEN=your_bot_token_here
BOT_OWNER_ID=123456789012345678        # Your Discord user ID

# ── SERVER ────────────────────────────────────────────────────────────────────
GUILD_ID=987654321098765432            # Your server ID (instant command sync)
                                       # Set to 0 for global sync (~1 hour delay)

# ── CONTAINER SETTINGS ────────────────────────────────────────────────────────
VPS_PREFIX=vps                         # Containers named: vps-username-1
DEFAULT_STORAGE_POOL=default           # Name from: lxc storage list

# ── RESOURCE CAPS ─────────────────────────────────────────────────────────────
MAX_RAM_GB=16
MAX_CPU_CORES=8

# ── COIN SYSTEM ───────────────────────────────────────────────────────────────
MAIN_CHAT_ID=0                         # Channel ID to earn coins by chatting
                                       # Set to 0 to disable chat earnings

# ── CLOUDFLARE (optional) ─────────────────────────────────────────────────────
ASM=False                              # Set True to auto-create subdomains
CF_API_TOKEN=                          # Cloudflare API token
CF_ZONE_ID=                            # Cloudflare zone ID
CF_DOMAIN=yourdomain.com               # Root domain for subdomains

# ── EXTENSIONS ────────────────────────────────────────────────────────────────
ACCOUNT=True
MARKETPLACE=True
ISTS=True                              # Ticket system
# All others default to False — toggle via /system in Discord
```

### How to get your Discord IDs

- **Bot Token:** [discord.com/developers](https://discord.com/developers) → Your App → Bot → Reset Token
- **User ID:** Discord → Settings → Advanced → Enable Developer Mode → right-click your name → Copy User ID
- **Server ID:** Right-click your server name → Copy Server ID
- **Channel ID:** Right-click a channel → Copy Channel ID

---

## 8. Run the Bot

```bash
# Direct run (for testing)
python3 bot.py
```

### With screen (keeps running after SSH disconnect)

```bash
screen -S moonbot
python3 bot.py
# Ctrl+A then D to detach
# screen -r moonbot  to reattach
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
User=youruser
WorkingDirectory=/home/youruser/moonnodes
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

# Follow logs live
journalctl -u moonbot -f
```

---

## 9. First Steps in Discord

After the bot starts you should see `Synced N command(s)` in the logs.

**Step 1 — Set up channels:**
```
/set setting:Log Channel         value:#bot-logs
/set setting:Ticket Channel      value:#support
/set setting:Payment Channel     value:#payments
```

**Step 2 — Add staff (optional):**
```
/role action:Add  user:@YourAdmin  permission:Admin
/role action:List
```

**Step 3 — Create VPS plans:**
```
/plans plan_type:Free     action:Add  plan_name:basic    ram_gb:1  cpu_cores:1  disk_gb:10  price:invite 3
/plans plan_type:Premium  action:Add  plan_name:starter  ram_gb:2  cpu_cores:2  disk_gb:20  price:₱150
```

**Step 4 — Add payment methods:**
```
/payment action:Add/Edit  payment_name:GCash  image:(attach QR image)
```

**Step 5 — Enable extensions:**
```
/system action:Extension Toggle  extension:Ticket System         ext_state:Enable
/system action:Extension Toggle  extension:Account & MoonCoin    ext_state:Enable
/system action:Extension Toggle  extension:Marketplace           ext_state:Enable
/system action:Extension Toggle  extension:Multi-Node Support    ext_state:Enable
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
| `/help` | Show commands available to your role |
| `/list` | View all your VPS instances and their status |
| `/manage` | Full VPS control panel — Start, Stop, Restart, SSH, Console, Reinstall |
| `/status` | Live host CPU, RAM, and Disk stats |
| `/plans plan_type:Free\|Premium` | Browse available VPS plans |
| `/claim` | Claim a Free VPS (invites or coins) or submit Premium payment proof |
| `/account options:Balance` | View your MoonCoin balance and multipliers |
| `/account options:VPS user_action:Share` | Grant or revoke VPS access for another user |
| `/account options:VPS user_action:Backup` | Create, list, restore, or delete VPS snapshots |
| `/backup` | Quick access to VPS backup actions |
| `/upgrade` | Upgrade or extend your VPS plan |
| `/support` | Open a private support ticket with staff |
| `/marketplace` | Buy, sell, and browse VPS listings |

### 🛡️ Admin Commands

| Command | Description |
|---|---|
| `/create` | Deploy a VPS for any user — pick node and OS |
| `/delete` | Permanently remove a user's VPS |
| `/manage user_id:` | Admin-view any user's VPS control panel |
| `/list action:All VPS` | View all VPS across every user |
| `/account admin_action:VPS vps_action:Suspend\|Unsuspend` | Suspend or unsuspend a VPS |
| `/account admin_action:Coin coin_action:Add\|Remove\|Reset\|View` | Manage user MoonCoins |
| `/account admin_action:VPS expire_action:Add\|Edit\|Remove\|View\|Clear` | Manage VPS expiry |
| `/set` | Configure log/ticket/payment channels and thresholds |
| `/payment` | Manage payment method images shown in `/plans` |
| `/giveaway action:Create` | Start a VPS giveaway with timer and auto-deploy |
| `/node` | Manage the LXD/Incus node cluster |
| `/role` | Assign, promote, demote, or list staff roles |

### 🔨 Mod Commands

| Command | Description |
|---|---|
| `/mod action:User Info target_user:` | Full user profile — VPS count, coins, join date |
| `/mod action:VPS Info container_name:` | Detailed container info by name |
| `/mod action:List VPS target_user:` | All VPS owned by a specific user |
| `/mod action:Search container_name:` | Find a VPS by partial container name |
| `/mod action:Server Stats` | Total VPS, users, running, suspended |
| `/mod action:Open Tickets` | List all open support threads |
| `/mod action:Suspension Logs` | Recent VPS suspension history |
| `/mod action:DM User target_user: message:` | Send a staff message to any user |
| `/mod action:Warn User target_user: message:` | Issue a formal warning (saved to DB) |
| `/mod action:Warn Log target_user:` | View a user's warning history |

### 🌟 Staff Commands

| Command | Description |
|---|---|
| `/staff action:My Tickets` | Support threads assigned to you |
| `/staff action:All Tickets` | All open support threads |
| `/staff action:Close Ticket` | Archive and lock a support thread |
| `/staff action:Claim Ticket` | Assign a ticket to yourself |
| `/staff action:Add Note` | Add a private staff note to a user |
| `/staff action:View Notes` | Read a user's staff notes |
| `/staff action:User Lookup` | Quick user profile summary |
| `/staff action:Warn User` | Issue a warning (Mod+) |
| `/staff action:Server Overview` | Global stats (Mod+) |
| `/staff action:Staff Announce` | Broadcast DM to all staff (Mod+) |
| `/staff action:Lock\|Unlock Channel` | Lock or unlock a channel (Admin+) |
| `/staff action:Purge` | Delete recent messages (Admin+) |

### 🌐 Node Commands

| Command | Description |
|---|---|
| `/node pick_one:List` | All registered nodes with host and port |
| `/node pick_one:Status option:` | Live CPU / RAM / Disk stats per node |
| `/node pick_one:Create name: host: port:` | Register a new remote node |
| `/node pick_one:Edit option:id` | Edit node details via modal |
| `/node pick_one:Delete option:id` | Remove a node from the cluster |
| `/node pick_one:Migrate vps_name: to_node:` | Move a VPS between nodes |

### 👑 Owner Commands

| Command | Description |
|---|---|
| `/system action:Extension Toggle` | Enable or disable any extension |
| `/system action:View Extensions` | See all on/off states |
| `/system action:Maintenance Mode` | Toggle maintenance mode instantly |
| `/system action:Clean DB` | Remove stale and empty VPS entries |
| `/system action:A2FA Management` | Set or remove your admin PIN |
| `/system action:MoonCoin Earning` | Add, edit, or remove coin earning rules |
| `/system action:Premium` | Assign or remove premium tiers to users |
| `/system action:List list_runtime:incus\|lxc list_node_id:` | List all containers on a node |

### 💻 Dev Commands

| Command | Description |
|---|---|
| `/dev action:Config Dump` | Show all runtime config values |
| `/dev action:DB Stats` | Row counts for all database tables |
| `/dev action:Runtime` | Container runtime binary path and version |
| `/dev action:Test LXC` | Run `lxc list` live and show output |
| `/dev action:Host Stats` | Live CPU, RAM, Disk of the host machine |
| `/dev action:Error Log` | Show the last 20 lines of bot.log |
| `/dev action:Reload Cog cog_name:` | Hot-reload a command module without restarting |
| `/dev action:Test DM` | Send a test DM to yourself |
| `/dev action:Extension State` | Show all extension on/off flags |
| `/dev action:Purge Cache` | Reload vps_data and admin_data from disk |

---

## 11. Extensions Reference

Toggle any extension at runtime — no bot restart required:

```
/system action:Extension Toggle  extension:<name>  ext_state:Enable|Disable
```

| Extension | What it does | Config key |
|---|---|---|
| **Multi-Node Support** | Deploy across an LXD/Incus cluster | `MULTI_NODE` |
| **Scheduled Backups** | Auto-snapshot all running containers on a schedule | `SOSB` |
| **Ticket System** | `/support` creates private Discord threads | `ISTS` |
| **Auto Expire & Suspend** | Suspend VPS automatically when expiry passes | `AES` |
| **Renewal Reminders** | DM users at 7, 3, and 1 day before expiry | `RR` |
| **Admin 2FA** | Require a PIN for sensitive admin actions | `A2FA` |
| **Multi-Server Tickets** | Separate ticket channels per Discord server | `MSTC` |
| **Backup Limits** | Cap max snapshots per VPS | `UBL_ENABLED` |
| **Marketplace** | Buy and sell VPS instances for coins or money | `MARKETPLACE` |
| **Account & MoonCoin** | `/account` and the full coin economy | `ACCOUNT` |
| **VPS Renewal** | Users can extend expiry with `/upgrade` | `VPS_RENEWAL` |
| **VPS Monitoring Alerts** | DM users when CPU / RAM exceeds threshold | `VPS_MONITORING` |
| **VPS Templates** | Minecraft, CS2, Nginx etc. in OS selector | `VPS_TEMPLATES` |
| **Multi-language** | i18n support (requires translation files) | `MULTILANG` |

---

## 12. Premium Tiers

Assign premium tiers to users:

```
/system action:Premium  premium_action:Set  premium_user:@User  premium_tier:pro
```

| Tier | Icon | Key |
|---|---|---|
| Basic | ⭐ | `basic` |
| Standard | 🌟 | `standard` |
| Pro | 💎 | `pro` |
| Elite | 🔱 | `elite` |

Premium users receive a tier badge in `/mod action:User Info` and `/account options:Balance`, plus a welcome DM when their tier is assigned.

---

## 13. MoonCoin Economy

MoonCoins are the in-bot currency used to claim free VPS plans and purchase from the marketplace.

### Default earning rules

| Action | Reward | Cooldown |
|---|---|---|
| Invite someone | +5 coins | None |
| Chat in main channel | +1 coin | 5 minutes |
| Enter a giveaway | +3 coins | Per giveaway |
| Join an event | +7 coins | Per event |

### Multipliers

- VIP Discord role → **+1%** on all earnings  
- Server Booster → **+2%** on all earnings  
- Multipliers stack with each other

### Admin coin management

```
/account admin_action:Coin  coin_action:Add       target_user:@User  transfer_amount:100
/account admin_action:Coin  coin_action:Remove    target_user:@User  transfer_amount:50
/account admin_action:Coin  coin_action:Reset     target_user:@User
/account admin_action:Coin  coin_action:View      target_user:@User
```

### Add custom earning rules

```
/system action:MoonCoin Earning  mooncoin_action:Add
  earning_name:daily  earning_amount:10  earning_cooldown:86400
```

---

## 14. File Structure

```
moonnodes/
├── bot.py                      # Entry point — loads cogs and background tasks
├── commands/
│   ├── Admin/
│   │   ├── set_cmd.py          # /set  /payment
│   │   └── vps.py              # /vps  (admin suspend/unsuspend)
│   ├── Dev/
│   │   └── dev_tools.py        # /dev  diagnostics and operations
│   ├── Mod/
│   │   └── mod_tools.py        # /mod  moderation tools
│   ├── Owner/
│   │   ├── admin.py            # /role  staff management
│   │   ├── create.py           # /create  VPS deployment
│   │   ├── delete.py           # /delete  VPS removal
│   │   ├── giveaway.py         # /giveaway
│   │   ├── nodes.py            # /node  cluster management
│   │   └── system.py           # /system  extensions, maintenance, owner tools
│   ├── Staff/
│   │   └── staff_tools.py      # /staff  ticket and moderation tools
│   └── User/
│       ├── account.py          # /account  coins, VPS sharing, expiry
│       ├── backup.py           # /backup  quick snapshot access
│       ├── claim.py            # /claim  free and premium VPS
│       ├── help.py             # /help  role-based command guide
│       ├── list_cmd.py         # /list  VPS instances
│       ├── manage.py           # /manage  VPS control panel
│       ├── marketplace.py      # /marketplace  buy and sell
│       ├── plans.py            # /plans  browse and manage plans
│       ├── status.py           # /status  host stats
│       ├── support.py          # /support  open a ticket
│       └── upgrade.py          # /upgrade  plan extension
├── core/
│   ├── config.py               # Loads .env — all runtime flags
│   ├── database.py             # SQLite helpers, async-safe lock, connection pool
│   ├── ext.py                  # Extension state checkers
│   ├── lxc.py                  # Router — detects Incus or LXD and delegates
│   ├── lxd.py                  # LXD backend (commands, deploy, stats)
│   ├── incus.py                # Incus backend (commands, deploy, stats)
│   ├── monitoring.py           # Host CPU / RAM / Disk monitoring thread
│   ├── cloudflare.py           # Cloudflare DNS subdomain automation
│   └── theme.py                # Embed colours, progress bars, shared builders
├── feature/
│   ├── codes/                  # Redemption codes system
│   └── plans/                  # Plans data and manager
├── ui/
│   ├── confirm_view.py         # Confirm / cancel dialog
│   ├── manage_view.py          # VPS control panel buttons
│   ├── node_select.py          # Node picker dropdown
│   ├── os_select.py            # OS picker dropdown (triggers deploy)
│   ├── plans_view.py           # Plans and giveaway entry buttons
│   └── software_view.py        # Software installer picker
├── vps_data.json               # VPS records (auto-created on first run)
├── admin_data.json             # Staff role data (auto-created)
├── moonnodes.db                # SQLite database (auto-created)
├── bot.log                     # Log file (auto-created)
├── .env                        # Your config — never commit this file
└── .env.example                # Template — copy to .env to get started
```

---

## 15. Troubleshooting

### "Synced 0 command(s)"

The bot connected but registered no slash commands.

```bash
# Ensure GUILD_ID is set correctly in .env
# Restart the bot after changing GUILD_ID
# Check bot.log for "Failed to load extension" errors
tail -50 bot.log | grep -i "error\|failed"
```

### "lxc: command not found" or "incus: command not found"

```bash
# LXD — add snap to PATH
export PATH=$PATH:/snap/bin
echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc
source ~/.bashrc

# Check incus
which incus
# If missing, re-run the install from section 4
```

### Bot cannot create containers (permission denied)

```bash
# LXD
sudo usermod -aG lxd $USER
# Log out and back in, or run:
newgrp lxd

# Incus
sudo usermod -aG incus-admin $USER
newgrp incus-admin

# Verify
lxc list       # or: incus list
```

### deploy_container — AttributeError: `__enter__`

`data_lock` in `core/database.py` is an `asyncio.Lock` and must be used with `async with`, not `with`. In `core/lxc.py`, both usages inside `deploy_container` must be:

```python
async with data_lock:
    ...
```

This was fixed in the latest version. If you still see the error, check that your `core/lxc.py` uses `async with data_lock` on lines ~250 and ~286.

### Containers stuck in "Starting"

```bash
# Check the runtime daemon
sudo systemctl status snap.lxd.daemon   # LXD
sudo systemctl status incus             # Incus

# Inspect container logs
lxc info --show-log <container-name>
incus info --show-log <container-name>
```

### SSH (tmate) not working

The bot installs `tmate` inside the container — the container needs internet access.

```bash
# Test from inside the container
lxc exec <container-name> -- ping -c 3 8.8.8.8

# If no network, check the bridge device in the profile
lxc profile show default | grep -A5 "devices"
```

If no `eth0` device:

```bash
# LXD
lxc network create lxdbr0
lxc profile device add default eth0 nic nictype=bridged parent=lxdbr0

# Incus
incus network create incusbr0
incus profile device add default eth0 nic nictype=bridged parent=incusbr0
```

### Multi-node — "connection refused" to remote node

```bash
# On the remote node — verify the listener is active
lxc config get core.https_address    # Expected: [::]:8443
incus config get core.https_address  # Expected: [::]:8443

# Check the firewall
sudo ufw status
sudo ufw allow 8443/tcp

# Test the API from the main server
curl -k https://REMOTE_IP:8443/1.0
# Expected: JSON response with "status":"Success"
```

### Database errors or corruption

```bash
# Review recent errors
tail -50 bot.log | grep -i "error\|warn\|corrupt"

# Backup first
cp moonnodes.db moonnodes.db.bak

# Rebuild the schema (preserves vps_data.json)
rm moonnodes.db
python3 -c "from core.database import init_db; init_db()"
```

### bot.log growing too large

```bash
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

*MoonNodes VPS Manager — built with discord.py and LXD / Incus*
