---
summary: "Install OpenClaw on a self-hosted Linux VM"
read_when:
  - You want to run OpenClaw on your own Linux server or VM
  - Setting up OpenClaw on a self-hosted infrastructure
  - Looking for a general Linux installation guide
title: "Self-Hosted Linux VM"
---

# Self-Hosted Linux VM

This guide covers installing OpenClaw on a self-hosted Linux virtual machine or server, whether that's running on your own hardware, a home lab, or a generic VPS provider.

## Goal

Run a persistent OpenClaw Gateway on your own Linux VM with remote access from your devices.

## Prerequisites

- Linux VM or server (Ubuntu 22.04+, Debian 11+, or similar)
- SSH access to the VM
- Root or sudo privileges
- ~30 minutes

## Supported Linux Distributions

OpenClaw works on most modern Linux distributions that support Node.js 22+:

- Ubuntu 22.04 LTS, 24.04 LTS
- Debian 11 (Bullseye), 12 (Bookworm)
- Fedora 38+
- Rocky Linux 9+
- Arch Linux (rolling)

## Quick Start

For experienced users who want to get running quickly:

```bash
# 1. Install Node.js 22+ (if not already installed)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# 2. Install OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash

# 3. Run onboarding
openclaw onboard --install-daemon

# 4. Access from your laptop (SSH tunnel)
ssh -N -L 18789:127.0.0.1:18789 user@your-vm-host

# 5. Open http://localhost:18789 and paste your token
```

## Step-by-Step Installation

### 1) Connect to Your VM

```bash
ssh user@your-vm-host
```

Replace `user` and `your-vm-host` with your actual username and hostname or IP address.

### 2) Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

For RedHat-based systems (Fedora, Rocky Linux):

```bash
sudo dnf update -y
```

### 3) Install Node.js 22+

OpenClaw requires Node.js 22 or newer.

#### Option A: Using NodeSource (Ubuntu/Debian)

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
```

#### Option B: Using NodeSource (Fedora/Rocky)

```bash
curl -fsSL https://rpm.nodesource.com/setup_22.x | sudo bash -
sudo dnf install -y nodejs
```

#### Option C: Using nvm (Any Distribution)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
```

Verify installation:

```bash
node -v  # Should show v22.x.x or higher
npm -v
```

### 4) Install Build Tools (Required for Native Modules)

```bash
# Ubuntu/Debian
sudo apt-get install -y build-essential python3

# Fedora/Rocky
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y python3
```

### 5) Install OpenClaw

Run the OpenClaw installer:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

The installer will:
- Install the `openclaw` CLI globally via npm
- Set up PATH if needed
- Run the onboarding wizard (optional)

If you want to skip onboarding during install:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
```

### 6) Run Onboarding

If you skipped onboarding or want to reconfigure:

```bash
openclaw onboard --install-daemon
```

The onboarding wizard will guide you through:
- Setting up authentication (API keys/tokens)
- Configuring messaging channels
- Installing the Gateway as a systemd service
- Testing your setup

When asked "How do you want to hatch your bot?", you can either:
- Configure channels now, or
- Select **"Do this later"** and configure via the Control UI

### 7) Configure Gateway for Remote Access

By default, the Gateway binds to loopback (localhost only). For remote access, you have several options:

#### Option A: SSH Tunnel (Recommended for Security)

Keep the Gateway on loopback and use an SSH tunnel:

```bash
# From your local machine
ssh -N -L 18789:127.0.0.1:18789 user@your-vm-host
```

Then access `http://localhost:18789` on your local machine.

#### Option B: Tailscale (Recommended for Easy Access)

Install Tailscale for secure remote access:

```bash
# On the VM
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname=openclaw-gateway

# Configure OpenClaw for Tailscale
openclaw config set gateway.tailscale.mode serve
openclaw config set gateway.trustedProxies '["127.0.0.1"]'
systemctl --user restart openclaw-gateway
```

Access from any device on your Tailscale network:
```
https://openclaw-gateway.<tailnet-name>.ts.net/
```

#### Option C: Direct Network Binding (Less Secure)

If you must bind to a network interface, use token authentication:

```bash
openclaw config set gateway.bind lan
openclaw config set gateway.auth.mode token
openclaw doctor --generate-gateway-token
systemctl --user restart openclaw-gateway
```

**Important:** If binding to `lan`, ensure your VM firewall only allows connections from trusted IPs.

### 8) Configure Systemd Service

The installer should have created a systemd user service. Verify it's running:

```bash
systemctl --user status openclaw-gateway
```

If you need to enable lingering (keeps services running after logout):

```bash
sudo loginctl enable-linger $USER
```

Common systemd commands:

```bash
# Check status
systemctl --user status openclaw-gateway

# Start/stop/restart
systemctl --user start openclaw-gateway
systemctl --user stop openclaw-gateway
systemctl --user restart openclaw-gateway

# View logs
journalctl --user -u openclaw-gateway -f
```

### 9) Verify Installation

```bash
# Check version
openclaw --version

# Check gateway health
openclaw health

# Check gateway status
openclaw status

# Run diagnostics
openclaw doctor
```

Test local access:

```bash
curl http://localhost:18789
```

## Firewall Configuration

If you're using a firewall on your VM, you'll need to configure it based on your access method:

### For SSH Tunnel (No Additional Firewall Rules Needed)

The Gateway stays on localhost, so no inbound firewall rules are needed beyond SSH (port 22).

### For Tailscale (Open UDP 41641)

```bash
# UFW
sudo ufw allow 41641/udp comment 'Tailscale'
sudo ufw enable

# firewalld
sudo firewall-cmd --permanent --add-port=41641/udp
sudo firewall-cmd --reload
```

### For Direct Network Access (Open Gateway Port)

Only do this if you understand the security implications:

```bash
# UFW - Allow from specific IP only
sudo ufw allow from YOUR_TRUSTED_IP to any port 18789 proto tcp
sudo ufw enable

# firewalld - Allow from specific IP only
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="YOUR_TRUSTED_IP" port port="18789" protocol="tcp" accept'
sudo firewall-cmd --reload
```

**Never expose the Gateway port to the entire internet without authentication!**

## Access the Control UI

Once the Gateway is running, access the Control UI:

### Via SSH Tunnel

1. Open tunnel: `ssh -N -L 18789:127.0.0.1:18789 user@your-vm-host`
2. Open `http://localhost:18789` in your browser
3. Paste the token shown during onboarding (or generate a new one with `openclaw doctor --generate-gateway-token`)

### Via Tailscale

1. Ensure Tailscale is set up on both the VM and your local device
2. Open `https://openclaw-gateway.<tailnet-name>.ts.net/`
3. Authenticate via Tailscale

## Security Best Practices

- **Keep the system updated:** Run `sudo apt update && sudo apt upgrade` regularly
- **Use SSH keys:** Disable password authentication in `/etc/ssh/sshd_config`
- **Use a firewall:** UFW, firewalld, or iptables
- **Bind to loopback:** Keep `gateway.bind loopback` unless you have a specific need
- **Use token auth:** Always set `gateway.auth.mode token` if binding to a network interface
- **Regular backups:** Back up `~/.openclaw/` directory
- **Run security audit:** `openclaw security audit`
- **Monitor logs:** `journalctl --user -u openclaw-gateway -f`

## Backup and Restore

### Backup

All OpenClaw data lives in `~/.openclaw/`:

```bash
# Create backup
tar -czvf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw

# Optional: backup to remote location
scp openclaw-backup-*.tar.gz user@backup-host:/path/to/backups/
```

### Restore

```bash
# Extract backup
tar -xzvf openclaw-backup-YYYYMMDD.tar.gz -C ~/

# Restart gateway
systemctl --user restart openclaw-gateway
```

## Updating OpenClaw

```bash
# Update to latest version
npm install -g openclaw@latest

# Run migration/repair if needed
openclaw doctor

# Restart gateway
systemctl --user restart openclaw-gateway

# Verify
openclaw health
openclaw --version
```

See [Updating](/install/updating) for more details.

## Troubleshooting

### Gateway won't start

```bash
# Check status
systemctl --user status openclaw-gateway

# View logs
journalctl --user -u openclaw-gateway -n 100

# Run diagnostics
openclaw doctor

# Try manual start to see errors
openclaw gateway run
```

### Can't access Control UI

```bash
# Verify gateway is listening
ss -tlnp | grep 18789

# Test local connection
curl http://localhost:18789

# Check firewall
sudo ufw status
sudo firewall-cmd --list-all

# Check SSH tunnel (if using)
ps aux | grep "ssh.*18789"
```

### `openclaw: command not found`

```bash
# Check npm global bin directory
npm prefix -g

# Add to PATH (add to ~/.bashrc or ~/.zshrc)
export PATH="$(npm prefix -g)/bin:$PATH"

# Reload shell
source ~/.bashrc
```

### Permission errors with systemd

```bash
# Enable lingering
sudo loginctl enable-linger $USER

# Verify lingering
loginctl show-user $USER | grep Linger
```

### Node.js version too old

```bash
# Check current version
node -v

# Install newer version (Ubuntu/Debian)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify
node -v  # Should be v22.x.x or higher
```

## Advanced Configuration

### Custom Gateway Port

```bash
openclaw config set gateway.port 8080
systemctl --user restart openclaw-gateway
```

### Multiple Profiles

Run multiple Gateway instances with different profiles:

```bash
# Create profile
openclaw config set --profile dev gateway.port 18790

# Install service for profile
openclaw gateway install --profile dev

# Manage profile service
systemctl --user start openclaw-gateway-dev
```

### Reverse Proxy (nginx)

If you want to put OpenClaw behind nginx:

```nginx
server {
    listen 80;
    server_name openclaw.example.com;

    location / {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout settings
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

Set trusted proxies:

```bash
openclaw config set gateway.trustedProxies '["127.0.0.1"]'
systemctl --user restart openclaw-gateway
```

## Common Use Cases

### Home Lab Server

Perfect for running on a dedicated home server or NAS with:
- 24/7 availability
- Local network access
- Optional VPN/Tailscale for remote access

### Cloud VPS (Generic Provider)

Works on any VPS provider (Linode, Vultr, DigitalOcean, etc.):
- Follow this guide for installation
- Use Tailscale for easy remote access
- See also: [Oracle Cloud](/platforms/oracle), [Hetzner](/platforms/hetzner), [GCP](/platforms/gcp)

### Development/Testing VM

Quick setup for testing:
- Use VM snapshot features for easy rollback
- Bind to loopback for security
- SSH tunnel for access

## See Also

- [Getting Started](/start/getting-started) — First-time setup guide
- [Gateway Configuration](/gateway/configuration) — All config options
- [Gateway Remote Access](/gateway/remote) — Remote access patterns
- [Platform-Specific Guides](/platforms) — Oracle, Hetzner, GCP, etc.
- [Docker Installation](/install/docker) — Alternative installation method
- [Updating](/install/updating) — How to update OpenClaw
- [VPS Hosting](/vps) — VPS provider comparison
