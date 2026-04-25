# 🌙 MoonNodes VPS Manager

Full-featured Discord bot for managing **LXD / Incus** VPS containers. All data in `system.db`.

---

## Quick Start

```bash
# System packages
apt update && apt upgrade -y
apt install -y python3 python3-pip git zip unzip curl screen

# Clone
git clone https://github.com/MoonLink-Team/MoonNode
cd MoonNode

# OR from zip
apt install zip
unzip dev.zip
cd MoonNode

# Python deps
pip3 install -r requirements.txt

# Config
cp .env.example .env
nano .env   # fill in DISCORD_TOKEN and BOT_OWNER_ID
```

---

## Install LXD (single node)

```bash
apt install snapd -y
snap install lxd --channel=latest/stable
lxd init --auto

# Add bot user to lxd group
usermod -aG lxd $USER
newgrp lxd

# Profile settings
lxc profile set default security.privileged true
lxc profile set default security.nesting true
lxc profile set default security.syscalls.intercept.mknod true

# Storage pool
lxc storage create default dir

# Network bridge
lxc network create lxdbr0
lxc profile device add default eth0 nic nictype=bridged parent=lxdbr0

# Test
lxc launch ubuntu:22.04 test && lxc list && lxc delete test --force
```

---

## Install Incus (single node, recommended)

```bash
# Install
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sh
incus admin init --auto

# Add bot user
usermod -aG incus-admin $USER
newgrp incus-admin

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

> **Root device note:** The bot runs `incus config device add <n> root disk path=/ pool=<pool> size=<N>GB` separately after init — the inline `--device` flag has parsing bugs in some Incus versions.

---

## Multi-Node — LXD

### On each remote node

```bash
snap install lxd --channel=latest/stable
lxd init --auto

lxc config set core.https_address [::]:8443
lxc config set core.trust_password "YourPassword"
ufw allow 8443/tcp

lxc storage create default dir
lxc profile set default security.privileged true
lxc profile set default security.nesting true
```

### From main server

```bash
lxc remote add node2 https://REMOTE_IP:8443 --accept-certificate
lxc list node2:   # verify
```

### Register in Discord

```
/node pick_one:Create  name:Node 2  runtime:lxc
/system action:Extension Toggle  extension:Multi-Node Support  ext_state:Enable
```

---

## Multi-Node — Incus

### On each remote node

```bash
curl -fsSL https://pkgs.zabbly.com/get/incus-stable | sh
incus admin init --auto

incus config set core.https_address [::]:8443
incus config trust add --name main-bot   # copy the token!
ufw allow 8443/tcp

incus storage create default dir
incus profile set default security.privileged true
incus profile set default security.nesting true
```

### From main server

```bash
incus remote add node2 https://REMOTE_IP:8443   # paste token
incus list node2:                               # verify
```

### Register in Discord

```
/node pick_one:Create  name:Node 2  runtime:incus
/system action:Extension Toggle  extension:Multi-Node Support  ext_state:Enable
```

---

## .env Reference

```env
# Required
DISCORD_TOKEN=your_bot_token
BOT_OWNER_ID=your_discord_user_id
GUILD_ID=your_server_id

# Container
VPS_PREFIX=vps
DEFAULT_STORAGE_POOL=default

# Limits
MAX_RAM_GB=16
MAX_CPU_CORES=8

# MoonCoin
MAIN_CHAT_ID=0   # channel ID for chat coin earning

# Role IDs (0 = not configured)
OWNER_ROLE_ID=0
HARD_ADMIN_ROLE_ID=0
ADMIN_ROLE_ID=0
MANAGER_ROLE_ID=0
HARD_MOD_ROLE_ID=0
MOD_ROLE_ID=0
HARD_STAFF_ROLE_ID=0
STAFF_ROLE_ID=0
TRIAL_STAFF_ROLE_ID=0
TRIAL_MOD_ROLE_ID=0
PARTNER_ROLE_ID=0
VIP_ROLE_ID=0

# Cloudflare (optional)
CF_API_TOKEN=
CF_ZONE_ID=
CF_DOMAIN=yourdomain.com

# Extensions (toggle via /system in Discord)
ACCOUNT=True
MARKETPLACE=True
ISTS=False
DM_COMMANDS=False
BLOCK_TERMINAL=False
```

---

## Run the Bot

```bash
# Direct
python3 bot.py

# Screen (persists after disconnect)
screen -S moonbot
python3 bot.py
# Ctrl+A D to detach | screen -r moonbot to reattach

# Systemd (recommended)
nano /etc/systemd/system/moonbot.service
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
systemctl daemon-reload
systemctl enable moonbot
systemctl start moonbot
systemctl status moonbot
journalctl -u moonbot -f
```

---

## First Boot

On first start the bot DMs you (the owner) asking if you have an existing `system.db`:

- **Upload system.db** → restores all previous data (VPS, coins, roles, settings)
- **Start Fresh** → clean database

---

## First Steps in Discord

```
/set setting:Log Channel         value:#logs
/set setting:Ticket Category     value:<CATEGORY_ID>
/set setting:Payment Channel     value:#payments
/set setting:Update Channel      value:#updates
/set setting:Claim Channel       value:#claims

/role action:Add  user:@Admin  permission:Admin

/plans plan_type:Free     action:Add  plan_name:basic   ram_gb:1  cpu_cores:1  disk_gb:10  price:coin 100
/plans plan_type:Premium  action:Add  plan_name:starter ram_gb:2  cpu_cores:2  disk_gb:20  price:₱150  validity:30 Days

/system action:Extension Toggle  extension:Ticket System       ext_state:Enable
/system action:Extension Toggle  extension:Account & MoonCoin  ext_state:Enable
/system action:Extension Toggle  extension:Marketplace         ext_state:Enable
```

---

## Update Bot

```bash
# Make sure it's a git repo first
cd /root/MoonNode
git init
git remote add origin https://github.com/MoonLink-Team/MoonNode
git fetch origin main
git reset --hard origin/main
```

Then in Discord:
```
/system action:🔄 Update Bot — Pull latest from GitHub
```

Bot auto-stashes local changes, pulls, unzips `dev.zip` if present, keeps `system.db` and `.env` safe. Then restart:

```bash
systemctl restart moonbot
```

---

## Transfer Data to New Server

```bash
# Stop bot on OLD server
systemctl stop moonbot

# Copy system.db to new server
scp /root/MoonNode/system.db root@NEW_IP:/root/MoonNode/system.db

# Start bot on NEW server
systemctl start moonbot
```

`system.db` = everything (VPS, coins, staff, settings, marketplace).

---

## Commands Reference

| Role | Commands |
|---|---|
| All Users | `/list` `/manage` `/status` `/plans` `/claim` `/account` `/support` `/marketplace` `/help` |
| Staff | `/staff role:Staff action:…` |
| Hard Staff | `/staff role:Hard Staff action:…` |
| Mod | `/mod role:Mod action:…` |
| Hard Mod | `/mod role:Hard Mod action:…` |
| Manager | All above + `/create` `/delete` `/role` `/set` `/payment` |
| Admin | All above + `/node` `/giveaway` `/system action:List` |
| Dev | `/dev action:…` |
| Owner | `/system action:…` (all) including Maintenance, Update Bot, Database |

---

## system.db Tables

| Table | Contents |
|---|---|
| `vps_entries` | All VPS records |
| `users` | Coins, invites, join dates |
| `admins` | Staff roles |
| `settings` | Channel IDs, config |
| `payments` | Payment method images |
| `marketplace` | User VPS listings |
| `dark_market` | Admin VPS listings |
| `warnings` | User warnings |
| `invite_tracking` | 7-day anti-fake invite |
| `claim_requests` | Premium claim submissions |
| `mooncoin_rules` | Earning rules |
| `blocked_commands` | Blocked VPS terminal commands |

---

## Troubleshooting

**"No root device could be found"** — Bot now adds root disk via separate `incus config device add` command. If still failing: `incus storage list` and set `DEFAULT_STORAGE_POOL=<name>` in `.env`.

**"Invalid device type disk,path=…"** — Fixed: bot no longer uses `--device` inline flag.

**"cannot pull with rebase: unstaged changes"** — Fixed: bot now stashes before pull and pops after.

**"AttributeError: __enter__"** — `data_lock` must be `asyncio.Lock`, use `async with data_lock`. Fixed in all files.

**Synced 0 commands** — Check `GUILD_ID` in `.env`. Restart bot after changing it.

**VPS no internet** — Bot auto-detects bridge (incusbr0, lxdbr0) and adds eth0 device before start. If still no internet: `incus network list` on host and check a bridge exists.
