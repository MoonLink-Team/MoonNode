# MoonNodes VPS Bot 🌙

A highly advanced, fully featured **LXC/LXD VPS Management System** built entirely within Discord. This bot allows users to deploy, manage, and connect to their own Linux containers directly from Discord via a sleek UI dashboard.

## ✨ Features

- **Live Discord Dashboard:** Real-time hardware metrics (CPU, RAM, Disk I/O) and server status.
- **Power Controls:** Start, Stop, or Reinstall OS natively through Discord buttons.
- **1-Click Software Installers:** Auto-install Docker, Game Servers (Minecraft, Rust, CS2, FiveM), Pterodactyl, WordPress, Traefik, Nginx, and more.
- **Secure Connections:** Automated Tailscale authentication and standard SSH connection strings.
- **Multi-Node Architecture:** Auto-balance and deploy containers across multiple physical dedicated servers.
- **Economy & Billing:** Sell Premium and Free VPS plans, integrated with GCash/PayPal proof of payments.
- **Automations:** Automated Cloudflare subdomains, daily snapshot backups, idle resource monitoring, and expiration suspensions.
- **Support System:** Built-in private ticket and thread creation linked to specific VPS instances.

## ⚙️ Requirements

- Ubuntu (20.04/22.04/24.04) or Debian (11/12) Host Server
- LXC / LXD hypervisor installed and initialized
- Python 3.8+
- A Discord Bot Token

## 🚀 Installation Guide

For the full step-by-step installation guide to prepare your Linux server, install LXD, and boot up the bot, please read the included [install.txt](install.txt) file.

### Quick Start
1. Clone the repository to your host machine.
2. Rename `.env.example` to `.env` and fill in your Discord Token and setup details.
3. Install dependencies:
   ```bash
   pip3 install discord.py python-dotenv aiohttp
   ```
4. Run the bot:
   ```bash
   python3 bot.py
   ```
# MoonNodes VPS Bot 🌙

A highly advanced, fully featured **LXC/LXD VPS Management System** built entirely within Discord. This bot allows users to deploy, manage, and connect to their own Linux containers directly from Discord via a sleek UI dashboard.

## ✨ Features

- **Live Discord Dashboard:** Real-time hardware metrics (CPU, RAM, Disk I/O) and server status.
- **Power Controls:** Start, Stop, or Reinstall OS natively through Discord buttons.
- **1-Click Software Installers:** Auto-install Docker, Game Servers (Minecraft, Rust, CS2, FiveM), Pterodactyl, WordPress, Traefik, Nginx, and more.
- **Secure Connections:** Automated Tailscale authentication and standard SSH connection strings.
- **Multi-Node Architecture:** Auto-balance and deploy containers across multiple physical dedicated servers.
- **Economy & Billing:** Sell Premium and Free VPS plans, integrated with GCash/PayPal proof of payments.
- **Automations:** Automated Cloudflare subdomains, daily snapshot backups, idle resource monitoring, and expiration suspensions.
- **Support System:** Built-in private ticket and thread creation linked to specific VPS instances.

## ⚙️ Requirements

- Ubuntu (20.04/22.04/24.04) or Debian (11/12) Host Server
- LXC / LXD hypervisor installed and initialized
- Python 3.8+
- A Discord Bot Token

## 🚀 Installation Guide

For the full step-by-step installation guide to prepare your Linux server, install LXD, and boot up the bot, please read the included [install.txt](install.txt) file.

### Quick Start
1. Clone the repository to your host machine.
2. Rename `.env.example` to `.env` and fill in your Discord Token and setup details.
3. Install dependencies:
   ```bash
   pip3 install discord.py python-dotenv aiohttp
   ```
4. Run the bot:
   ```bash
   python3 bot.py
   ```
