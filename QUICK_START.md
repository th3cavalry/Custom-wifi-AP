# Quick Start Guide - Dell Precision T7810 Home Server

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Dell Precision T7810 with 2x Xeon E5-2620 v3, 32GB RAM, 500GB HDD
- [ ] All hardware components ordered (see HARDWARE_GUIDE.md)
- [ ] USB drive (8GB+) for Ubuntu installation
- [ ] Ethernet cable for initial setup
- [ ] Monitor and keyboard for initial configuration
- [ ] Internet access for downloading software

## Hardware Installation (1-2 hours)

### Step 1: Prepare the System

1. **Power down** completely and unplug from power
2. **Ground yourself** to prevent static damage
3. **Open the case** (remove side panel)

### Step 2: Install Components

**In this order:**

1. **NVMe SSD** (for OS)
   - Install M.2 to PCIe adapter in Slot 3 (PCIe x4)
   - Insert NVMe SSD into adapter
   - Secure with included screw
   - Attach heatsink if provided

2. **WiFi 7 Card**
   - Install in Slot 1 (PCIe x16)
   - Secure card with bracket screw
   - Connect antennas to card
   - Route antenna cables through case slot
   - Mount external antennas

3. **RM520 5G Modem**
   - Install M.2 Key B to PCIe adapter in Slot 2 (PCIe x4)
   - Insert RM520 module into adapter
   - Connect 4x antenna cables
   - Route cables through case
   - Mount external 5G antennas
   - Insert SIM card into modem

4. **Additional Storage** (for LANCache)
   - Install 4TB SSD in 2.5" drive bay, OR
   - Install 4TB HDD in 3.5" drive bay
   - Connect SATA data cable
   - Connect SATA power cable

5. **Optional: Network Card**
   - Install Intel I350-T4 in Slot 4
   - Secure card

### Step 3: Cable Management

- Organize cables for proper airflow
- Secure loose cables with zip ties
- Ensure no cables block fans

### Step 4: Final Check

- [ ] All cards properly seated
- [ ] All power connections secure
- [ ] Antennas connected
- [ ] SIM card inserted
- [ ] Case closed

### Step 5: First Boot

1. **Connect**:
   - Monitor to DisplayPort/DVI
   - Keyboard to USB
   - Ethernet cable to onboard NIC

2. **Power on** and enter BIOS (F2 during boot)

3. **BIOS Configuration**:
   - Update BIOS if needed (check Dell support)
   - Boot mode: UEFI
   - Disable Secure Boot
   - Enable Virtualization (VT-x, VT-d)
   - Set boot priority: NVMe first
   - Save and exit

## Software Installation (2-3 hours)

### Step 1: Install Ubuntu Server

1. **Create bootable USB**:
   ```bash
   # On Linux/Mac:
   sudo dd if=ubuntu-24.04-server.iso of=/dev/sdX bs=4M status=progress
   
   # On Windows: Use Rufus or similar tool
   ```

2. **Boot from USB** (F12 during boot for boot menu)

3. **Installation wizard**:
   - Language: English
   - Update installer: Yes
   - Keyboard: Your layout
   - Install type: Ubuntu Server
   - Network: Configure Ethernet (skip WiFi)
   - Proxy: None (unless required)
   - Mirror: Default
   - Storage:
     - Use entire NVMe disk
     - Set up LVM: Yes
     - Partition scheme: Guided
   - Profile:
     - Your name: `admin` (or preferred)
     - Server name: `homeserver`
     - Username: `admin`
     - Password: **Strong password**
   - SSH: Enable OpenSSH server
   - Featured snaps: None
   - Complete installation

4. **Reboot** and remove USB drive

### Step 2: Initial System Configuration (15 minutes)

```bash
# Log in as your user

# Update system
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y git curl wget htop iotop net-tools \
  build-essential vim nano screen tmux

# Set timezone
sudo timedatectl set-timezone America/New_York  # Adjust to your timezone

# Reboot if kernel updated
sudo reboot
```

### Step 3: Configure RM520 Modem (30 minutes)

```bash
# Install modem tools
sudo apt install -y modemmanager network-manager \
  libqmi-utils libmbim-utils

# Enable ModemManager
sudo systemctl enable ModemManager
sudo systemctl start ModemManager

# Wait 30 seconds for modem detection
sleep 30

# Check modem
sudo mmcli -L
sudo mmcli -m 0

# Configure connection (replace with your APN)
sudo nmcli connection add \
  type gsm \
  ifname '*' \
  con-name cellular \
  apn "your.provider.apn" \
  connection.autoconnect yes

# Activate
sudo nmcli connection up cellular

# Verify
ip addr show wwan0
ping -c 4 8.8.8.8
```

**Common APNs:**
- T-Mobile: `fast.t-mobile.com`
- AT&T: `broadband`
- Verizon: `vzwinternet`

### Step 4: Configure WiFi AP (30 minutes)

```bash
# Install hostapd
sudo apt install -y hostapd

# Stop service during config
sudo systemctl stop hostapd
sudo systemctl unmask hostapd

# Create config
sudo nano /etc/hostapd/hostapd.conf
```

**Minimal config** (paste this):
```ini
interface=wlan0
driver=nl80211
ssid=YourHomeNetwork
hw_mode=a
channel=36
ieee80211ac=1
ieee80211ax=1
wpa=2
wpa_key_mgmt=SAE
wpa_passphrase=YourSecurePassword123
rsn_pairwise=CCMP
country_code=US
wmm_enabled=1
```

```bash
# Enable service
sudo nano /etc/default/hostapd
# Add: DAEMON_CONF="/etc/hostapd/hostapd.conf"

# Configure network
sudo nano /etc/netplan/01-netcfg.yaml
```

**Add to netplan**:
```yaml
network:
  version: 2
  renderer: networkd
  wifis:
    wlan0:
      dhcp4: no
      addresses:
        - 192.168.1.1/24
```

```bash
# Apply
sudo netplan apply

# Enable and start
sudo systemctl enable hostapd
sudo systemctl start hostapd

# Check status
sudo systemctl status hostapd
```

### Step 5: Setup Routing and NAT (15 minutes)

```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Install iptables
sudo apt install -y iptables-persistent

# Configure NAT (replace wwan0 if different)
sudo iptables -t nat -A POSTROUTING -o wwan0 -j MASQUERADE
sudo iptables -A FORWARD -i wlan0 -o wwan0 -j ACCEPT
sudo iptables -A FORWARD -i wwan0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Save rules
sudo netfilter-persistent save

# Verify
sudo iptables -t nat -L -n -v
```

### Step 6: Install AdGuard Home (20 minutes)

```bash
# Download and install
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v

# Access web interface
# From another device connected to WiFi:
# http://192.168.1.1:3000
```

**Initial setup via web UI:**
1. Click "Get Started"
2. Admin interface: Keep port 3000
3. DNS port: 53
4. Create admin username/password
5. Configure upstream DNS:
   - `https://dns.cloudflare.com/dns-query`
   - `https://dns.google/dns-query`
6. Enable DNSSEC
7. Test configuration
8. Finish setup

### Step 7: Setup DHCP (10 minutes)

```bash
# Install dnsmasq
sudo apt install -y dnsmasq
sudo systemctl stop dnsmasq

# Configure
sudo nano /etc/dnsmasq.conf
```

**Add to config**:
```ini
port=0
interface=wlan0
dhcp-range=192.168.1.50,192.168.1.250,24h
dhcp-option=option:router,192.168.1.1
dhcp-option=option:dns-server,192.168.1.1
```

```bash
# Start service
sudo systemctl enable dnsmasq
sudo systemctl start dnsmasq
sudo systemctl status dnsmasq
```

### Step 8: Install Docker and LANCache (30 minutes)

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install -y docker-compose-plugin

# Logout and login again
exit
# SSH back in

# Verify
docker --version
docker compose version

# Prepare cache storage
sudo mkdir -p /cache/lancache-data
sudo mkdir -p /cache/lancache-logs
sudo chown -R $USER:$USER /cache

# Create LANCache directory
mkdir -p ~/lancache
cd ~/lancache

# Create docker-compose.yml
nano docker-compose.yml
```

**Paste this config**:
```yaml
version: '3'

services:
  lancache:
    image: lancachenet/monolithic:latest
    container_name: lancache
    restart: unless-stopped
    environment:
      - CACHE_ROOT=/data/cache
      - CACHE_DISK_SIZE=3500g
      - CACHE_MAX_AGE=3650d
      - CACHE_INDEX_SIZE=500m
    volumes:
      - /cache/lancache-data:/data/cache
      - /cache/lancache-logs:/data/logs
    ports:
      - "80:80"
      - "443:443"

  lancache-dns:
    image: lancachenet/lancache-dns:latest
    container_name: lancache-dns
    restart: unless-stopped
    environment:
      - USE_GENERIC_CACHE=true
      - LANCACHE_IP=192.168.1.1
      - UPSTREAM_DNS=127.0.0.1#5335
    ports:
      - "127.0.0.1:5380:53/udp"
      - "127.0.0.1:5380:53/tcp"
```

```bash
# Start LANCache
docker compose up -d

# Check status
docker compose ps
docker compose logs -f
```

### Step 9: Configure DNS Integration (10 minutes)

In AdGuard Home web UI (http://192.168.1.1:3000):

1. Go to **Settings** → **DNS settings**
2. Scroll to **Upstream DNS servers**
3. Click **Add upstream DNS**
4. Add these conditionals:

```
[/steamcontent.com/]127.0.0.1:5380
[/epicgames.com/]127.0.0.1:5380
[/blizzard.com/]127.0.0.1:5380
[/ea.com/]127.0.0.1:5380
[/origin.com/]127.0.0.1:5380
[/ubisoft.com/]127.0.0.1:5380
[/windowsupdate.com/]127.0.0.1:5380
```

5. Click **Save**

### Step 10: Install Monitoring (Optional, 10 minutes)

```bash
# Install Netdata
bash <(curl -Ss https://my-netdata.io/kickstart.sh)

# Access at: http://192.168.1.1:19999
```

## Testing and Verification (30 minutes)

### Test 1: Internet Connection

```bash
# From server
ping -c 4 8.8.8.8
ping -c 4 google.com

# Check cellular stats
sudo mmcli -m 0 | grep signal
```

### Test 2: WiFi Connection

1. Connect device to "YourHomeNetwork"
2. Check IP address (should be 192.168.1.x)
3. Test internet: Browse to google.com
4. Check DNS: `nslookup google.com`

### Test 3: AdGuard

1. Open http://192.168.1.1:3000
2. Check Dashboard for query stats
3. Try accessing an ad domain (should be blocked)
4. Visit: http://adblocktest.com (should show blocked ads)

### Test 4: LANCache

```bash
# Check containers
docker ps

# From client device, test DNS:
nslookup steamcontent.com
# Should return 192.168.1.1

# Test HTTP:
curl -I http://192.168.1.1
# Should return cache server response

# Monitor cache:
watch -n 5 'du -sh /cache/lancache-data'
```

### Test 5: Performance

```bash
# Speed test from server
sudo apt install -y speedtest-cli
speedtest-cli

# WiFi speed test from client
# Use speedtest.net or fast.com
```

## Quick Reference Commands

### System Management

```bash
# System status
sudo systemctl status ModemManager hostapd dnsmasq AdGuardHome

# Reboot
sudo reboot

# Shutdown
sudo shutdown -h now
```

### Modem Commands

```bash
# Modem info
sudo mmcli -m 0

# Signal strength
sudo mmcli -m 0 | grep signal

# Restart connection
sudo nmcli connection down cellular
sudo nmcli connection up cellular
```

### WiFi Commands

```bash
# Restart WiFi AP
sudo systemctl restart hostapd

# Check connected clients
sudo iw dev wlan0 station dump | grep Station

# View hostapd logs
sudo journalctl -u hostapd -f
```

### LANCache Commands

```bash
# Container status
docker ps
docker compose ps

# View logs
docker compose logs -f lancache

# Restart
docker compose restart

# Cache size
du -sh /cache/lancache-data
```

### Network Diagnostics

```bash
# Check interfaces
ip addr show

# Check routes
ip route show

# Check connections
ss -tulpn

# Network traffic
sudo iftop

# DNS test
dig @192.168.1.1 google.com
nslookup google.com 192.168.1.1
```

## Troubleshooting Quick Fixes

### No Internet

```bash
# Check modem
sudo mmcli -L
sudo mmcli -m 0

# Restart connection
sudo systemctl restart ModemManager
sudo nmcli connection up cellular
```

### WiFi Not Working

```bash
# Check service
sudo systemctl status hostapd

# Check config
sudo nano /etc/hostapd/hostapd.conf

# Restart
sudo systemctl restart hostapd

# Check logs
sudo journalctl -u hostapd -n 50
```

### Slow Performance

```bash
# Check system load
htop

# Check disk I/O
sudo iotop

# Check network usage
sudo iftop

# Check cache disk
df -h /cache
```

### LANCache Not Working

```bash
# Check containers
docker ps

# Restart containers
cd ~/lancache
docker compose restart

# Check DNS
dig @127.0.0.1 -p 5380 steamcontent.com

# View logs
docker compose logs -f
```

## Next Steps

1. ✅ Complete basic setup (you're here!)
2. 📊 Monitor performance for 24-48 hours
3. 🔧 Fine-tune settings based on usage
4. 📱 Configure static IPs for important devices
5. 🛡️ Review security settings
6. 💾 Set up automated backups
7. 📚 Read full guides:
   - [HARDWARE_GUIDE.md](HARDWARE_GUIDE.md)
   - [SOFTWARE_SETUP.md](SOFTWARE_SETUP.md)
   - [NETWORK_ARCHITECTURE.md](NETWORK_ARCHITECTURE.md)

## Support and Resources

- **Issues?** Check troubleshooting section above
- **Questions?** Review the detailed guides
- **Community**: r/homelab, r/selfhosted
- **Documentation**: 
  - AdGuard: https://github.com/AdguardTeam/AdGuardHome/wiki
  - LANCache: https://lancache.net/docs/

## Security Reminders

- [ ] Change default passwords
- [ ] Enable firewall (see SOFTWARE_SETUP.md)
- [ ] Disable SSH password auth (use keys)
- [ ] Keep system updated
- [ ] Regular backups
- [ ] Monitor logs for anomalies

## Congratulations!

Your Dell Precision T7810 is now a fully functional home server with:
- ✅ 5G cellular internet connection
- ✅ WiFi 7 access point
- ✅ Ad blocking with AdGuard Home
- ✅ Game caching with LANCache
- ✅ Full routing and NAT

Enjoy your new home network!
