# 🌙 MoonNodes VPS Manager — Full Setup & Command Guide

---

## 📋 TABLE OF CONTENTS
1. [System Requirements](#system-requirements)
2. [Server Setup (Bash)](#server-setup)
3. [Bot Installation](#bot-installation)
4. [Environment Configuration (.env)](#environment-configuration)
5. [Running the Bot](#running-the-bot)
6. [First-Time Discord Setup](#first-time-discord-setup)
7. [All Slash Commands](#all-slash-commands)
8. [Feature Flags Reference](#feature-flags-reference)

---

## 1. System Requirements

| Requirement | Minimum |
|---|---|
| OS | Ubuntu 20.04+ / Debian 11+ |
| Python | 3.10+ |
| LXC | 4.0+ |
| RAM | 2GB+ (host) |
| Storage | 20GB+ |
| Discord.py | 2.3+ |

---

## 2. Server Setup (Bash)

### Step 1 — Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2 — Install Python & pip
```bash
sudo apt install python3 python3-pip python3-venv -y
python3 --version   # Should be 3.10+
```

### Step 3 — Install LXC
```bash
sudo apt install lxc lxc-utils -y
lxc --version       # Confirm install
```

### Step 4 — Install Git & clone the bot
```bash
sudo apt install git -y
git clone https://github.com/MoonLink-Team/MoonNode
cd MoonNode
```

### Step 5 — Create a virtual environment
```bash
python3 -m venv venv
source venv/bin/activate
```

### Step 6 — Install Python dependencies
```bash
pip install discord.py python-dotenv
```

### Step 7 — Verify LXC storage pool
```bash
lxc storage list
# Note the pool name — you'll use it in .env as DEFAULT_STORAGE_POOL
```

### Step 8 — (Optional) Create a dedicated LXC storage pool
```bash
lxc storage create vpspool dir source=/var/lib/lxc/vpspool
lxc storage list   # Confirm it appears
```

### Step 9 — (Optional) Run as a systemd service
```bash
sudo nano /etc/systemd/system/moonnodes.service
```
Paste this into the file:
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
Then enable and start it:
```bash
sudo systemctl daemon-reload
sudo systemctl enable moonnodes
sudo systemctl start moonnodes
sudo systemctl status moonnodes   # Should say "active (running)"
```

### Useful systemd commands
```bash
sudo systemctl stop moonnodes          # Stop the bot
sudo systemctl restart moonnodes       # Restart the bot
sudo journalctl -u moonnodes -f        # Live logs
sudo journalctl -u moonnodes -n 100    # Last 100 log lines
```

---

## 3. Bot Installation

### Create a Discord Application
1. Go to https://discord.com/developers/applications
2. Click **New Application** → give it a name
3. Go to **Bot** tab → click **Add Bot**
4. Under **Privileged Gateway Intents**, enable:
   - **Server Members Intent**
   - **Message Content Intent**
5. Copy your **Bot Token** (you'll need this for `.env`)
6. Go to **OAuth2 → URL Generator**:
   - Scopes: `bot`, `applications.commands`
   - Bot Permissions: `Administrator`
7. Copy the generated URL and open it to invite the bot to your server

---

## 4. Environment Configuration (.env)

Create a `.env` file in the project root:

```bash
nano .env
```

Paste and fill in:

```env
# ───────────────────────────────────────────────
# REQUIRED
# ───────────────────────────────────────────────
DISCORD_TOKEN=your_bot_token_here
BOT_OWNER_ID=your_discord_user_id_here

# ───────────────────────────────────────────────
# APPEARANCE
# ───────────────────────────────────────────────
BOT_COLOR=5865F2           # Hex color for embeds (no #)

# ───────────────────────────────────────────────
# LXC / VPS
# ───────────────────────────────────────────────
VPS_PREFIX=vps             # Prefix for container names (e.g. vps-username-1)
DEFAULT_STORAGE_POOL=      # LXC pool name from: lxc storage list
VPS_USER_ROLE_ID=0         # Discord role ID to grant after VPS creation (0 = disabled)

# ───────────────────────────────────────────────
# RESOURCE LIMITS
# ───────────────────────────────────────────────
MAX_RAM_GB=16              # Max RAM a single VPS can have
MAX_CPU_CORES=8            # Max CPU cores a single VPS can have
CPU_THRESHOLD=90           # % CPU usage to trigger alert
RAM_THRESHOLD=90           # % RAM usage to trigger alert
ENFORCE_RESOURCE_LIMITS=True

# ───────────────────────────────────────────────
# CLAIMS
# ───────────────────────────────────────────────
PLANS=True                 # Enable /plans command
CLAIM_PREMIUM=True         # Allow premium VPS claims
CLAIM_FREE=True            # Allow free VPS claims

# ───────────────────────────────────────────────
# FEATURE FLAGS (True / False)
# ───────────────────────────────────────────────
SOSB=False         # Scheduled automated backups (runs every 24h)
AES=False          # Auto-expire & suspend VPS on expiry date
RR=False           # Renewal reminder DMs (3 days before expiry)
ABEI=False         # Economy/credit system
RA=False           # Referral affiliate tracking
CPCS=False         # Promo code system
UBL_ENABLED=False  # Enforce per-plan backup limits
BACKUP_LIMIT=1     # Default backup limit (if UBL_ENABLED=True)
ASM=False          # Advanced server monitoring
Multi-Node=False   # Multi-node LXC cluster support
ISTS=False         # Instant server template system
MSTC=False         # Multi-server ticket channel
PUDC=False         # Per-user disk cap
A2FA=False         # Admin 2-Factor Authentication PIN

# ───────────────────────────────────────────────
# CLOUDFLARE (optional — for subdomain automation)
# ───────────────────────────────────────────────
CF_API_TOKEN=
CF_ZONE_ID=
CF_DOMAIN=yourdomain.com
HOST_PUBLIC_IP=
```

---

## 5. Running the Bot

### Manually
```bash
source venv/bin/activate
python bot.py
```

### With systemd (recommended)
```bash
sudo systemctl start moonnodes
```

### Check logs
```bash
# Systemd logs
sudo journalctl -u moonnodes -f

# Or run directly and watch terminal output
python bot.py
```

---

## 6. First-Time Discord Setup

After the bot is online in your server, run these slash commands **as the Bot Owner**:

### Set channels
```
/set setting:Log Channel       value:<channel_id>
/set setting:Payment Channel   value:<channel_id>
/set setting:Ticket Channel    value:<channel_id>
```

### Set resource alert thresholds
```
/set setting:CPU Threshold %   value:90
/set setting:RAM Threshold %   value:90
```

### Add staff roles
```
/admin action:Add permission:Owner  user:@yourself
/admin action:Add permission:Admin  user:@youradmin
/admin action:Add permission:Mod    user:@yourmod
```

### Add payment methods
```
/payment action:Add/Edit  payment_name:GCash    image:(attach QR image)
/payment action:Add/Edit  payment_name:PayPal   image:(attach QR image)
```

### Add VPS plans
```
# Free plans
/plans plan_type:Free    action:Add (Admin)  plan_name:basic     ram_gb:1  cpu_cores:1  disk_gb:10  price:Free
/plans plan_type:Free    action:Add (Admin)  plan_name:invite4   ram_gb:2  cpu_cores:1  disk_gb:15  price:invite 4

# Premium plans
/plans plan_type:Premium action:Add (Admin)  plan_name:starter   ram_gb:2  cpu_cores:2  disk_gb:20  price:₱150   validity:30 Days
/plans plan_type:Premium action:Add (Admin)  plan_name:pro       ram_gb:4  cpu_cores:4  disk_gb:40  price:₱300   validity:30 Days
/plans plan_type:Premium action:Add (Admin)  plan_name:ultimate  ram_gb:8  cpu_cores:8  disk_gb:80  price:₱500   validity:30 Days
```

---

## 7. All Slash Commands

---

### 👤 USER COMMANDS

#### `/help`
Shows all available commands.

---

#### `/list`
View all your active VPS instances — shows status, config, OS, node, and expiry.

---

#### `/manage`
Open the VPS control panel (start, stop, restart, rebuild, console, etc.)

| Option | Description |
|---|---|
| `user` | (Admin only) Manage another user's VPS |

---

#### `/manage-shared`
Manage a VPS that another user shared with you.

| Option | Description |
|---|---|
| `owner` | The VPS owner (Discord member) |
| `vps_number` | Which VPS (1, 2, 3…) |

---

#### `/backup`
Create or restore a VPS snapshot.

| Option | Values |
|---|---|
| `action` | Create Backup / List Backups / Restore Latest |
| `vps_number` | Which VPS (default: 1) |

---

#### `/share`
Grant another Discord user access to your VPS.

| Option | Description |
|---|---|
| `user` | Discord member to share with |
| `vps_number` | Which VPS (default: 1) |

---

#### `/plans`
Browse or manage VPS plans.

| Option | Values |
|---|---|
| `plan_type` | Premium / Free |
| `action` | List / Add (Admin) / Edit (Admin) / Remove (Admin) |
| `plan_name` | e.g. basic, pro, starter |
| `ram_gb` | RAM in GB |
| `cpu_cores` | CPU cores |
| `disk_gb` | Disk in GB |
| `price` | e.g. ₱150 or `invite 4` |
| `validity` | e.g. `30 Days` |
| `emoji` | Emoji icon for the plan |

---

#### `/claim`
Claim a VPS or redeem a promo code.

| Option | Values |
|---|---|
| `claim_type` | Premium / Free / Promo Code |
| `code_or_plan` | Plan name (e.g. basic) or promo code |
| `proof` | Payment screenshot (required for Premium) |

---

#### `/affiliate`
View your referral credits or transfer credits to another user.

| Option | Description |
|---|---|
| `target_user` | User to view or transfer to |
| `transfer_amount` | Amount of credits |
| `admin_action` | (Admin only) Add Credits / Remove Credits |

---

#### `/transfer`
Sell or gift your VPS to another user.

| Option | Description |
|---|---|
| `user` | Recipient |
| `vps_number` | Which VPS (default: 1) |

---

#### `/upgrade`
Upgrade your VPS plan using credits.

| Option | Description |
|---|---|
| `vps_number` | Which VPS (default: 1) |
| `plan_name` | Target plan name |

---

#### `/support`
Open a support ticket.

---

#### `/status`
View live host node resource stats (CPU, RAM, disk).

---

### 🛡️ ADMIN / OWNER COMMANDS

#### `/admin`
Assign or remove staff roles. (Owner only)

| Option | Values |
|---|---|
| `action` | Add / Remove / List |
| `permission` | Owner Level / Admin Level / Mod Level / Dev Level |
| `user` | Target Discord member |

---

#### `/set`
Configure bot settings.

| Option | Values |
|---|---|
| `setting` | Log Channel / Payment Channel / Ticket Channel / CPU Threshold % / RAM Threshold % |
| `value` | Channel ID or number |

---

#### `/payment`
Manage payment methods shown in `/plans`.

| Option | Values |
|---|---|
| `action` | Add/Edit / Remove / List |
| `payment_name` | Name of the method (e.g. GCash) |
| `image` | QR code or payment image |

---

#### `/create`
Deploy a new VPS for a user. (Walks through Node → OS selection UI)

| Option | Description |
|---|---|
| `user` | Target Discord member |
| `plan_name` | Plan to deploy (e.g. starter) |
| `plan_type` | Premium or Free |

---

#### `/delete`
Permanently delete a user's VPS container.

| Option | Description |
|---|---|
| `user` | VPS owner |
| `vps_number` | Which VPS (1, 2…) |
| `reason` | Reason for deletion |

---

#### `/vps`
Suspend or unsuspend a VPS. (Admin)

| Option | Values |
|---|---|
| `action` | Suspend VPS / Unsuspend VPS |
| `container_name` | Exact container name |
| `reason` | Reason |

---

#### `/giveaway`
Start or manage a VPS giveaway.

| Option | Values |
|---|---|
| `action` | Create / Cancel / List |
| (additional options per action) | Plan, duration, winners, etc. |

---

#### `/nodes`
View multi-node cluster status with live CPU, RAM, and disk usage per node.

---

#### `/list-all`
View all VPS instances across all users and nodes.

---

#### `/system-admin`
System-level toggles, tools, and maintenance. (Owner only)

| Option | Values |
|---|---|
| `action` | Extension Toggle / Maintenance Mode / Migrate VPS / Clean DB / View Extensions / A2FA Management |
| `extension` | asm / multi_node / sosb / abei / ists / aes / rr / ra / a2fa / cpcs / mstc / pudc / ubl |
| `ext_state` | Enable / Disable |
| `vps_name` | Container name (for Migrate) |
| `target_node` | Destination node (for Migrate) |
| `a2fa_action` | Edit PIN / Remove PIN |
| `new_pin` | 4-digit PIN |

---

## 8. Feature Flags Reference

| Flag | .env Key | What it does |
|---|---|---|
| Scheduled Backups | `SOSB=True` | Auto-snapshot all VPS every 24h |
| Auto Expire & Suspend | `AES=True` | Stop VPS when expiry date passes |
| Renewal Reminders | `RR=True` | DM users 3 days before expiry |
| Economy / Credits | `ABEI=True` | Enable credit balance system |
| Referral Affiliate | `RA=True` | Track referrals, earn credits |
| Promo Codes | `CPCS=True` | Enable `/claim` promo code redemption |
| Backup Limits | `UBL_ENABLED=True` | Cap backups per plan tier |
| Multi-Node | `Multi-Node=True` | Enable multi-LXC-node cluster |
| Admin 2FA PIN | `A2FA=True` | Require PIN for sensitive admin actions |

---

*MoonNodes VPS Manager — Bot Guide*
