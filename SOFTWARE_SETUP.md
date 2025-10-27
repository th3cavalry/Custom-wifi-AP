# Dell Precision T7810 Home Server Software Setup Guide

## Overview

This guide covers the complete software setup for your Dell Precision T7810 home server with:
- 5G cellular modem (RM520) as WAN connection
- WiFi 7 access point
- AdGuard Home for DNS filtering and ad blocking
- LANCache for game content caching
- Network management and monitoring

## Recommended Operating System

### Option 1: Ubuntu Server 24.04 LTS (Recommended)

**Advantages:**
- Long-term support (until 2029)
- Excellent hardware support
- Large community and documentation
- Easy package management
- Native Docker support

**Download:** https://ubuntu.com/download/server

### Option 2: Proxmox VE 8.x (For Advanced Users)

**Advantages:**
- Virtualization platform
- Can run multiple VMs/containers
- Better resource isolation
- Web-based management
- Can run all services in separate containers

**Download:** https://www.proxmox.com/en/downloads

This guide will focus on **Ubuntu Server 24.04 LTS** for simplicity.

## Initial System Setup

### 1. Install Ubuntu Server

1. Create bootable USB with Ubuntu Server 24.04
2. Boot from USB and follow installation wizard
3. **Storage configuration**:
   - NVMe SSD: OS and system (100GB partition + rest for Docker)
   - 4TB SSD: Mount as `/cache` for LANCache
   - Existing HDD: Additional storage or backup

4. **Network configuration during install**:
   - Configure ethernet interface for initial setup
   - Skip WiFi configuration (will configure later)

5. **User setup**:
   - Create admin user
   - Enable OpenSSH server for remote management

6. **Complete installation and reboot**

### 2. Post-Installation System Update

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y \
  build-essential \
  git \
  curl \
  wget \
  htop \
  iotop \
  net-tools \
  ethtool \
  bridge-utils \
  vlan \
  dnsutils \
  tcpdump \
  iperf3

# Reboot if kernel was updated
sudo reboot
```

### 3. Update System Firmware and Drivers

```bash
# Install fwupd for firmware updates
sudo apt install -y fwupd

# Check for firmware updates
sudo fwupdmgr refresh
sudo fwupdmgr get-updates
sudo fwupdmgr update
```

## Hardware Configuration

### 1. RM520 5G Modem Setup

#### Install ModemManager and Dependencies

```bash
# Install modem management tools
sudo apt install -y \
  modemmanager \
  libmbim-utils \
  libqmi-utils \
  network-manager

# Enable and start ModemManager
sudo systemctl enable ModemManager
sudo systemctl start ModemManager

# Check modem detection
sudo mmcli -L
sudo mmcli -m 0  # View modem details
```

#### Configure Modem Connection

```bash
# Create connection using nmcli
sudo nmcli connection add \
  type gsm \
  ifname '*' \
  con-name cellular \
  apn "your.provider.apn" \
  connection.autoconnect yes

# Example APNs:
# T-Mobile: fast.t-mobile.com
# AT&T: broadband
# Verizon: vzwinternet

# Activate connection
sudo nmcli connection up cellular

# Check connection status
nmcli connection show cellular
ip addr show wwan0  # or device name shown by ModemManager
```

#### Create Failover Script (Optional)

Create `/usr/local/bin/modem-watchdog.sh`:

```bash
#!/bin/bash
# Modem connection watchdog

INTERFACE="wwan0"  # Adjust based on your modem interface
CONNECTION="cellular"
PING_HOST="8.8.8.8"

while true; do
    if ! ping -c 3 -W 5 -I $INTERFACE $PING_HOST > /dev/null 2>&1; then
        echo "$(date): Connection down, restarting..."
        nmcli connection down $CONNECTION
        sleep 5
        nmcli connection up $CONNECTION
    fi
    sleep 60
done
```

Make executable and create systemd service:
```bash
sudo chmod +x /usr/local/bin/modem-watchdog.sh
```

### 2. WiFi 7 Access Point Setup

#### Install hostapd

```bash
# Install hostapd and dependencies
sudo apt install -y hostapd

# Stop hostapd during configuration
sudo systemctl stop hostapd
sudo systemctl unmask hostapd
```

#### Configure hostapd

Create/edit `/etc/hostapd/hostapd.conf`:

```ini
# Interface configuration
interface=wlan0
driver=nl80211

# Network name (SSID)
ssid=YourHomeNetwork

# WiFi 7 configuration (802.11be)
hw_mode=a
channel=36  # 5GHz channel, adjust based on region
ieee80211ac=1
ieee80211ax=1
ieee80211be=1  # Enable WiFi 7

# Security (WPA3)
wpa=2
wpa_key_mgmt=SAE
wpa_passphrase=YourSecurePassword
rsn_pairwise=CCMP

# Additional settings
country_code=US  # Adjust to your country
wmm_enabled=1
auth_algs=1

# Performance settings
he_su_beamformer=1
he_su_beamformee=1
he_mu_beamformer=1

# 6GHz band support (if available)
#op_class=131
#he_oper_chwidth=2
#he_oper_centr_freq_seg0_idx=15

# Enable 160MHz channel width for maximum performance
he_oper_chwidth=2
vht_oper_chwidth=2
vht_oper_centr_freq_seg0_idx=42
```

#### Enable and Start hostapd

```bash
# Edit /etc/default/hostapd
sudo nano /etc/default/hostapd
# Add line: DAEMON_CONF="/etc/hostapd/hostapd.conf"

# Enable and start
sudo systemctl enable hostapd
sudo systemctl start hostapd

# Check status
sudo systemctl status hostapd
```

### 3. Network Bridge and Routing Setup

#### Configure Network Interfaces

Edit `/etc/netplan/01-netcfg.yaml`:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    # WAN interface (cellular modem)
    wwan0:
      dhcp4: yes
      optional: true
      
    # LAN interface (if using additional NIC)
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.1/24
      
  wifis:
    wlan0:
      dhcp4: no
      addresses:
        - 192.168.1.1/24
      access-points:
        "YourHomeNetwork":
          mode: ap
```

Apply configuration:
```bash
sudo netplan apply
```

#### Enable IP Forwarding and NAT

```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# Make persistent
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf

# Configure iptables for NAT
sudo apt install -y iptables-persistent

# Setup NAT rules
sudo iptables -t nat -A POSTROUTING -o wwan0 -j MASQUERADE
sudo iptables -A FORWARD -i wlan0 -o wwan0 -j ACCEPT
sudo iptables -A FORWARD -i wwan0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Save rules
sudo netfilter-persistent save
```

### 4. DHCP Server Setup

```bash
# Install dnsmasq for DHCP (we'll disable DNS since using AdGuard)
sudo apt install -y dnsmasq
sudo systemctl stop dnsmasq
```

Edit `/etc/dnsmasq.conf`:

```ini
# Disable DNS (AdGuard will handle this)
port=0

# DHCP configuration
interface=wlan0
dhcp-range=192.168.1.50,192.168.1.250,24h
dhcp-option=option:router,192.168.1.1
dhcp-option=option:dns-server,192.168.1.1  # Points to AdGuard

# Static leases (optional)
#dhcp-host=AA:BB:CC:DD:EE:FF,192.168.1.100
```

```bash
sudo systemctl enable dnsmasq
sudo systemctl start dnsmasq
```

## AdGuard Home Setup

### 1. Install AdGuard Home

```bash
# Download and install
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v

# The installer will:
# - Install AdGuard Home
# - Create systemd service
# - Start the service
```

### 2. Initial Configuration

1. Open web browser: `http://192.168.1.1:3000`
2. Follow setup wizard:
   - Set admin interface port (keep default 3000 or change)
   - Set DNS server port: 53
   - Create admin username and password
3. Configure DNS settings:
   - **Upstream DNS servers**:
     ```
     https://dns.cloudflare.com/dns-query
     https://dns.google/dns-query
     tls://1.1.1.1
     ```
   - **Bootstrap DNS**: `1.1.1.1`, `8.8.8.8`
   - Enable DNSSEC
   - Enable parallel queries

### 3. Configure Filters

Add these filter lists in Settings → DNS blocklists:

1. **AdGuard DNS filter**: Pre-configured
2. **AdAway Default**: `https://adaway.org/hosts.txt`
3. **EasyList**: Pre-configured
4. **Malware/Phishing**: `https://urlhaus.abuse.ch/downloads/hostfile/`
5. **Steven Black's hosts**: `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts`

### 4. Configure DNS for LANCache Integration

In Settings → DNS settings → DNS rewrites, add:

```
lancache.steamcontent.com → 192.168.1.1
lancache.epicgames.com → 192.168.1.1
# (More specific entries will be added by LANCache)
```

## LANCache Setup

### 1. Install Docker and Docker Compose

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install -y docker-compose-plugin

# Verify installation
docker --version
docker compose version
```

### 2. Prepare LANCache Storage

```bash
# Create mount point for cache drive
sudo mkdir -p /cache/lancache-data
sudo mkdir -p /cache/lancache-logs

# Set permissions
sudo chown -R $USER:$USER /cache

# Check available space
df -h /cache
```

### 3. Install LANCache with Docker

Create directory for LANCache:

```bash
mkdir -p ~/lancache
cd ~/lancache
```

Create `docker-compose.yml`:

```yaml
version: '3'

services:
  lancache:
    image: lancachenet/monolithic:latest
    container_name: lancache
    restart: unless-stopped
    environment:
      - CACHE_ROOT=/data/cache
      - CACHE_DISK_SIZE=3500g  # Adjust based on your cache drive size
      - CACHE_MAX_AGE=3650d
      - CACHE_INDEX_SIZE=500m
      - TZ=America/New_York  # Adjust to your timezone
    volumes:
      - /cache/lancache-data:/data/cache
      - /cache/lancache-logs:/data/logs
    ports:
      - "80:80"
      - "443:443"
    networks:
      - lancache

  lancache-dns:
    image: lancachenet/lancache-dns:latest
    container_name: lancache-dns
    restart: unless-stopped
    environment:
      - USE_GENERIC_CACHE=true
      - LANCACHE_IP=192.168.1.1  # Your server IP
      - UPSTREAM_DNS=127.0.0.1#5335  # Forward to AdGuard on different port
    ports:
      - "127.0.0.1:5380:53/udp"
      - "127.0.0.1:5380:53/tcp"
    networks:
      - lancache

networks:
  lancache:
    driver: bridge
```

### 4. Configure DNS Integration

We need to forward gaming-related DNS queries to LANCache DNS:

Edit AdGuard Home configuration:
1. Go to Settings → DNS settings
2. Add upstream DNS for specific domains:

```
[/steamcontent.com/]127.0.0.1:5380
[/epicgames.com/]127.0.0.1:5380
[/blizzard.com/]127.0.0.1:5380
[/ea.com/]127.0.0.1:5380
[/origin.com/]127.0.0.1:5380
[/ubisoft.com/]127.0.0.1:5380
[/windowsupdate.com/]127.0.0.1:5380
```

### 5. Start LANCache

```bash
cd ~/lancache
docker compose up -d

# Check status
docker compose ps
docker compose logs -f lancache
```

### 6. Monitor LANCache

Check cache statistics:
```bash
# View logs
docker compose logs -f

# Check cache size
du -sh /cache/lancache-data

# Monitor in real-time
watch -n 5 'du -sh /cache/lancache-data && docker stats --no-stream lancache'
```

## System Monitoring and Management

### 1. Install Monitoring Tools

```bash
# Install Netdata for system monitoring
bash <(curl -Ss https://my-netdata.io/kickstart.sh)

# Access at: http://192.168.1.1:19999
```

### 2. Setup Automatic Updates

```bash
# Install unattended-upgrades
sudo apt install -y unattended-upgrades

# Configure
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 3. Create Management Scripts

Create `/usr/local/bin/server-status.sh`:

```bash
#!/bin/bash
echo "=== System Status ==="
echo "Uptime: $(uptime -p)"
echo "Load: $(uptime | awk -F'load average:' '{print $2}')"
echo ""
echo "=== Network Status ==="
echo "Cellular: $(nmcli -t -f STATE,SIGNAL con show cellular | head -1)"
echo "WiFi Clients: $(iw dev wlan0 station dump | grep Station | wc -l)"
echo ""
echo "=== Service Status ==="
echo "AdGuard: $(systemctl is-active AdGuardHome)"
echo "hostapd: $(systemctl is-active hostapd)"
echo "LANCache: $(docker inspect -f '{{.State.Status}}' lancache 2>/dev/null || echo 'not running')"
echo ""
echo "=== Cache Status ==="
echo "Cache Size: $(du -sh /cache/lancache-data 2>/dev/null | cut -f1)"
echo "Cache Free: $(df -h /cache | tail -1 | awk '{print $4}')"
```

Make executable:
```bash
sudo chmod +x /usr/local/bin/server-status.sh
```

## Network Optimization

### 1. Optimize TCP Settings

Add to `/etc/sysctl.conf`:

```ini
# TCP performance tuning
net.core.rmem_max=134217728
net.core.wmem_max=134217728
net.ipv4.tcp_rmem=4096 87380 67108864
net.ipv4.tcp_wmem=4096 65536 67108864
net.core.netdev_max_backlog=5000
net.ipv4.tcp_congestion_control=bbr
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_mtu_probing=1
```

Apply:
```bash
sudo sysctl -p
```

### 2. WiFi Optimization

Edit `/etc/hostapd/hostapd.conf` and add:

```ini
# QoS and performance
he_default_pe_duration=4
he_twt_required=0
he_rts_threshold=1023

# Enable beamforming
he_su_beamformer=1
he_su_beamformee=1
he_mu_beamformer=1

# Multi-user settings
he_bss_color=3
he_mu_edca_ac_be_aifsn=8
he_mu_edca_ac_be_ecwmin=9
he_mu_edca_ac_be_ecwmax=10
```

## Security Hardening

### 1. Configure Firewall

```bash
# Install UFW
sudo apt install -y ufw

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (change port if using non-standard)
sudo ufw allow 22/tcp

# Allow DNS (AdGuard)
sudo ufw allow 53/tcp
sudo ufw allow 53/udp

# Allow DHCP
sudo ufw allow 67/udp

# Allow HTTP/HTTPS (LANCache)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow AdGuard web interface (limit to LAN)
sudo ufw allow from 192.168.1.0/24 to any port 3000

# Allow Netdata (limit to LAN)
sudo ufw allow from 192.168.1.0/24 to any port 19999

# Enable firewall
sudo ufw enable
```

### 2. Secure SSH

Edit `/etc/ssh/sshd_config`:

```ini
PermitRootLogin no
PasswordAuthentication no  # Use SSH keys only
PubkeyAuthentication yes
X11Forwarding no
```

Restart SSH:
```bash
sudo systemctl restart sshd
```

### 3. Enable Fail2ban

```bash
sudo apt install -y fail2ban

# Start and enable
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

## Backup and Recovery

### 1. Create Backup Script

Create `/usr/local/bin/backup-config.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup configurations
tar -czf $BACKUP_DIR/config_$DATE.tar.gz \
  /etc/hostapd/ \
  /etc/netplan/ \
  /etc/dnsmasq.conf \
  ~/lancache/docker-compose.yml \
  /opt/AdGuardHome/AdGuardHome.yaml

# Keep only last 7 backups
ls -t $BACKUP_DIR/config_*.tar.gz | tail -n +8 | xargs -r rm

echo "Backup completed: $BACKUP_DIR/config_$DATE.tar.gz"
```

Schedule with cron:
```bash
sudo crontab -e
# Add: 0 2 * * * /usr/local/bin/backup-config.sh
```

## Troubleshooting

### Modem Issues

```bash
# Check modem status
sudo mmcli -m 0

# Reset modem
sudo mmcli -m 0 --reset

# Check signal strength
sudo mmcli -m 0 | grep signal

# Manual connection test
sudo qmicli -d /dev/cdc-wdm0 --wds-start-network="apn='your.apn'"
```

### WiFi Issues

```bash
# Check hostapd status
sudo systemctl status hostapd
sudo journalctl -u hostapd -f

# Test WiFi interface
sudo iw dev wlan0 info

# Check connected clients
sudo iw dev wlan0 station dump
```

### LANCache Issues

```bash
# Check container status
docker compose ps
docker compose logs lancache

# Test cache hit
curl -I http://192.168.1.1/

# Check cache disk usage
docker exec lancache df -h /data/cache
```

### DNS Issues

```bash
# Test DNS resolution
dig @192.168.1.1 google.com
nslookup google.com 192.168.1.1

# Check AdGuard logs
tail -f /opt/AdGuardHome/data/querylog.json

# Test LANCache DNS
dig @127.0.0.1 -p 5380 steamcontent.com
```

## Performance Monitoring

### Create Dashboard Script

Create `/usr/local/bin/dashboard.sh`:

```bash
#!/bin/bash
clear
while true; do
  clear
  echo "╔══════════════════════════════════════════════════════════════╗"
  echo "║        Dell T7810 Home Server Dashboard                      ║"
  echo "╚══════════════════════════════════════════════════════════════╝"
  echo ""
  
  # System info
  echo "System: $(uptime -p) | Load: $(cat /proc/loadavg | awk '{print $1,$2,$3}')"
  echo "CPU: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}')% | RAM: $(free -h | awk '/^Mem/ {print $3"/"$2}')"
  echo ""
  
  # Network
  echo "Cellular: $(mmcli -m 0 2>/dev/null | grep 'signal quality' | awk '{print $3}') | State: $(mmcli -m 0 2>/dev/null | grep 'state:' | awk '{print $2}')"
  echo "WiFi Clients: $(iw dev wlan0 station dump 2>/dev/null | grep Station | wc -l)"
  echo ""
  
  # Services
  echo "Services:"
  echo "  AdGuard: $(systemctl is-active AdGuardHome) | hostapd: $(systemctl is-active hostapd)"
  echo "  LANCache: $(docker inspect -f '{{.State.Status}}' lancache 2>/dev/null || echo 'stopped')"
  echo ""
  
  # Cache stats
  echo "LANCache: $(du -sh /cache/lancache-data 2>/dev/null | awk '{print $1}') / $(df -h /cache 2>/dev/null | tail -1 | awk '{print $4}') free"
  echo ""
  
  echo "Press Ctrl+C to exit"
  sleep 5
done
```

Make executable:
```bash
chmod +x /usr/local/bin/dashboard.sh
```

## Maintenance Schedule

### Daily
- Monitor system performance via Netdata
- Check cellular signal strength
- Verify all services running

### Weekly
- Review AdGuard query logs
- Check LANCache hit rates
- Monitor disk usage
- Review system logs

### Monthly
- Update system packages
- Update Docker images
- Review and rotate logs
- Test backup restoration
- Clean old cache entries if needed

### Quarterly
- Update firmware (system BIOS, modem, WiFi card)
- Review and optimize configurations
- Check hardware health
- Update documentation

## Additional Resources

- **AdGuard Home**: https://github.com/AdguardTeam/AdGuardHome
- **LANCache**: https://lancache.net/
- **hostapd**: https://w1.fi/hostapd/
- **ModemManager**: https://www.freedesktop.org/wiki/Software/ModemManager/
- **Ubuntu Server**: https://ubuntu.com/server/docs

## Next Steps

1. Complete hardware installation (see HARDWARE_GUIDE.md)
2. Follow this guide step-by-step
3. Test each component before proceeding
4. Document any custom changes
5. Set up monitoring and alerts
6. Create a maintenance log
