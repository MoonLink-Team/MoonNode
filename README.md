# MoonNodes VPS Manager

An enterprise-grade Discord bot to automate, manage, and scale LXC/LXD VPS hosting operations.

## 🚀 Core Features
- **Automated Deployments:** Deploy Ubuntu/Debian containers directly via Discord commands.
- **Resource Management:** Set RAM, CPU, and Disk limitations. Auto-suspends over-utilization.
- **Game Server Auto-Installers:** One-click install for Minecraft, Rust, CS2, and FiveM via the Software Installer menu.
- **Economy & Billing (ABEI):** Built-in credit system with promo codes (`CPCS`), upgrades/downgrades (`PUDC`), and a user marketplace (`MSTC`).
- **Multi-Node Architecture:** Deploy instances across multiple physical LXD remotes with smart auto-balancing.
- **Automated Subdomains (ASM):** Cloudflare integration for instant domain generation.

## User Commands
* `/list` - View your active VPS instances.
* `/manage` - Open the control panel to start/stop, reinstall, open Web SSH, or install software.
* `/backup` - Create or restore a snapshot of your VPS (Subject to User Backup Limits).
* `/share` - Add or remove access to your VPS for another Discord user.
* `/plans` - View available Free and Premium plans.
* `/claim` - Claim a Premium VPS, a Free VPS, or redeem a Promo Code.
* `/affiliate` - Check your referral credits, or transfer credits to another user.
* `/support` - Open a private ticket thread with the staff regarding a specific VPS.
* `/status` - Check the host node's live CPU/RAM and uptime.
* `/transfer` - (If MSTC is enabled) Transfer your VPS to another user for credits.
* `/upgrade` - (If PUDC is enabled) Upgrade your VPS plan using your credit balance.

## Admin Commands
* `/system-admin` - Manage system toggles, clean the DB, edit A2FA, or migrate VPS containers between nodes.
* `/vps` - Force suspend or unsuspend a user's VPS.
* `/create` - Bypass restrictions and forcefully deploy a VPS to a user.
* `/delete` - Terminate a VPS and delete all user data permanently.
* `/giveaway` - Start a Discord giveaway where the winner automatically receives a deployed VPS.
* `/list-all` - View all running and stopped VPS instances across the network.
* `/admin` - Add or remove Owner, Admin, Mod, or Dev roles.
* `/set` - Change log channels, ticket channels, and auto-shutdown resource limits.
* `/nodes` - Check the status of the multi-node cluster.

## ⚙️ Extensions (`/system-admin Extension`)
1. **asm** - Automated Subdomain Management
2. **multi_node** - Multi-Node Architecture & Migrations
3. **sosb** - Scheduled & Off-Site Backups
4. **abei** - Automated Billing & Economy
5. **ists** - Integrated Support Ticket System
6. **aes** - Automated Expiration & Suspensions
7. **rr** - Renewal Reminders
8. **ra** - Referral / Affiliate System
9. **a2fa** - Admin 2-Factor Authentication
10. **cpcs** - Coupon & Promo Code System
11. **mstc** - Marketplace & Server Transfers
12. **pudc** - Plan Upgrades & Downgrades
13. **ubl** - User Backup Limits
