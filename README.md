# 🌙 MoonNodes VPS Manager Bot

A full-featured Discord bot for managing Incus/LXC VPS containers — with KVM/QEMU support, MoonCoin economy, AF2F admin 2-factor verification, animated responses, and multi-node infrastructure.

---

## ✨ What's New (Latest Update)

| Area | Change |
|---|---|
| 🖥️ **KVM/QEMU** | Full VM support via `--vm` flag in Incus — pick LXC or KVM on `/create` |
| 🌙 **Debian** | Debian 11, 12, 13 fully supported (containers + VMs) |
| 🔐 **AF2F** | Admin 2-Factor verification for sensitive commands |
| 🎬 **Animations** | Named animation sets in `animation/` — deploy, startup, coins, more |
| ⏳ **Expire** | `/mod role::trident: action:Expire` — set/edit/remove VPS expiry |
| 🔒 **Command Lock** | All commands locked until bot owner completes first-boot setup |
| 🚀 **First Boot** | 10-option setup menu sent to owner on first start |
| 🌙 **MoonCoin** | `/set setting:MoonCoin` — add/remove/view rules, grant/deduct coins |
| 🔧 **Network Fix** | Post-start eth0 verification + automatic NIC re-attach if missing |
| 📝 **DM Commands** | Renamed from "Can DM Commands" → "DM Commands" |
| 🗑️ `/account` | Admin actions removed — moved to `/set` (coins) and `/mod` (VPS) |

---

## 📋 Requirements

- Python 3.11+
- Incus (preferred) or LXD installed on host
- Discord bot token
- KVM acceleration for VM mode (`/dev/kvm` must exist)

```bash
pip install -r requirements.txt
```

---

## 🚀 First Boot

On first run the bot **locks all commands** for non-owners and DMs the owner a setup menu with 10 options:

| Option | Description |
|---|---|
| 📤 Restore system.db | Upload existing database — all VPS/user data restored |
| ✅ Start Fresh | Clean slate, configure from scratch |
| ⚙️ Quick Setup | Fresh start with step-by-step channel guide |
| 🔗 Restore from URL | Paste a direct download link to system.db |
| 🌐 Multi-Node Setup | Fresh start, configure remote nodes next |
| 📂 Import vps_data.json | Migrate legacy JSON backup to new DB format |
| 🎭 Demo Mode | Explore commands safely without real deployments |
| ⚡ Minimal Mode | Core VPS only, extensions off by default |
| ⏭️ Skip | Start immediately, configure later |
| 🔄 Copy from Another Bot | Instructions to clone another instance |

Commands unlock for everyone once the owner clicks any button.

---

## 🔐 AF2F — Admin 2-Factor Verification

Sensitive commands require a **4-digit code** from the AF2F channel before executing.

**Bot owner (`BOT_OWNER_ID`) is always exempt.**

### Protected Commands

- `/create` /delete` `/set` `/system` `/node` `/role`
- `/mod role::trident: action:Expire`
- `/mod role::trident: action:Suspend VPS`
- `/mod role::trident: action:Unsuspend VPS`

### Setup

```
/set setting:AF2F Channel value:<channel_id>
```

Only the bot owner can configure this. The bot posts codes there when staff trigger protected commands.

### Flow

1. Staff member runs a protected command
2. Bot sends 4-digit code to the AF2F channel (owner sees it)
3. Staff member runs `/af2f code:<CODE>` within 5 minutes
4. Command proceeds

---

## 🖥️ KVM/QEMU Virtual Machines

### Usage

```
/create user:@username vm_mode:KVM/QEMU VM
```

### LXC vs KVM

| Feature | LXC Container | KVM/QEMU VM |
|---|---|---|
| Boot time | ~5 seconds | ~30–60 seconds |
| Isolation | Kernel shared | Full hardware virtualisation |
| Windows support | ❌ | ✅ |
| FreeBSD support | Limited | ✅ |
| Performance | ⚡ Near-native | Slight overhead |
| KVM required | ❌ | ✅ `/dev/kvm` needed |

### Host Requirements

```bash
ls /dev/kvm          # must exist
incus info | grep vm # Incus must support VMs
```

---

## 🌙 MoonCoin

| Action | Command |
|---|---|
| View rules | `/set setting:MoonCoin mooncoin_action:View` |
| Add rule | `/set setting:MoonCoin mooncoin_action:Add value:<name> option:<amount>` |
| Remove rule | `/set setting:MoonCoin mooncoin_action:Remove value:<name>` |
| Grant coins | `/set setting:MoonCoin mooncoin_action:Grant target_user:@user value:<amount>` |
| Deduct coins | `/set setting:MoonCoin mooncoin_action:Deduct target_user:@user value:<amount>` |

> MoonCoin **transfer** animations are intentionally excluded by design.

---

## 🎬 Animations

Add `animation/{name}.py` with a `FRAMES = [...]` list to create a new animation.

Built-in sets: `loading`, `deploy`, `startup`, `mooncoin_earn`, `mooncoin_spend`, `mooncoin_view`, `suspend`, `af2f`, `first_boot`.

`mooncoin_transfer` is intentionally absent.

---

## ⏳ VPS Expiry

```
/mod role::trident: action:Expire expire_action:Add container_name:vps-user-1 expiry_time:30d
/mod role::trident: action:Expire expire_action:Edit expiry_time:2026-06-01
/mod role::trident: action:Expire expire_action:Remove
/mod role::trident: action:Expire expire_action:View target_user:@user
```

🔐 AF2F required for all Expire actions (non-owner staff).

---

## 🌐 Supported OS

**LXC + KVM:** Ubuntu 20.04/22.04/24.04, Debian 11/12/13, CentOS Stream 9, AlmaLinux 9, Rocky Linux 9, Fedora 39, Alpine Linux

**KVM only:** Windows Server 2022, FreeBSD 14

---

## 🔧 Network Fix

If a container starts with no internet (`Network is unreachable`), the bot now automatically:
1. Checks `eth0` inside the container post-start
2. Re-attaches NIC to the detected bridge if missing
3. Restarts the container

Manual fix on host:
```bash
incus network create incusbr0
incus config device add <container> eth0 nic nictype=bridged parent=incusbr0
incus restart <container>
```

---

## 📁 Structure

```
├── bot.py
├── animation/           # Named animation frame sets
├── commands/
│   ├── Owner/
│   │   ├── create.py    # /create (LXC + KVM)
│   │   └── af2f_cmd.py  # /af2f verify
│   ├── Admin/
│   │   └── set_cmd.py   # /set + MoonCoin + AF2F channel
│   └── Mod/
│       └── mod_tools.py # /mod + Expire + AF2F gates
├── core/
│   ├── af2f.py          # AF2F core logic
│   ├── incus.py         # Incus (LXC + KVM/QEMU)
│   └── lxc.py           # deploy_container() wrapper
└── ui/
    ├── node_select.py
    └── os_select.py     # Animated deploy
```

---

## 📜 Changelog

### v3.0.0
- KVM/QEMU VM support
- AF2F 2-factor system
- Animated deploy flow
- First-boot 10-option menu + command lock
- `/mod action:Expire` for Hard Mod
- `/set setting:MoonCoin` full coin management
- Debian normalization + full OS support
- Post-start eth0 NIC auto-recovery
- `animation/` extensible package
- `/account` admin panel removed (→ `/set` + `/mod`)
- DB: `mooncoin_rules`, `af2f_pending`, `vps_expiry_overrides` tables
- VPS entries now include `type: container|vm`
