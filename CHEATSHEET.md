# Quick Reference Cheat Sheet

## Essential Commands

### System Status
```bash
# Overall system status
/usr/local/bin/server-status.sh

# Service status
sudo systemctl status ModemManager hostapd dnsmasq AdGuardHome

# Check all running containers
docker ps

# System resources
htop
sudo iotop
```

### Modem Commands
```bash
# List modems
sudo mmcli -L

# Detailed modem info
sudo mmcli -m 0

# Signal strength
sudo mmcli -m 0 | grep signal

# Connection status
nmcli connection show cellular

# Restart connection
sudo nmcli connection down cellular && sudo nmcli connection up cellular

# Modem reset
sudo mmcli -m 0 --reset
```

### WiFi Commands
```bash
# Restart WiFi AP
sudo systemctl restart hostapd

# Check WiFi status
sudo systemctl status hostapd

# View connected clients
sudo iw dev wlan0 station dump

# Count connected clients
sudo iw dev wlan0 station dump | grep Station | wc -l

# Check interface status
iw dev wlan0 info

# View logs
sudo journalctl -u hostapd -f
```

### Network Commands
```bash
# Show all interfaces
ip addr show

# Show routing table
ip route show

# Show listening ports
sudo ss -tulpn

# Network traffic monitor
sudo iftop

# Test DNS
dig @192.168.1.1 google.com
nslookup google.com 192.168.1.1

# Ping test
ping -c 4 8.8.8.8

# Speed test
speedtest-cli
```

### AdGuard Commands
```bash
# Service status
sudo systemctl status AdGuardHome

# Restart service
sudo systemctl restart AdGuardHome

# View logs
sudo journalctl -u AdGuardHome -f

# Configuration file
sudo nano /opt/AdGuardHome/AdGuardHome.yaml

# Web interface
http://192.168.1.1:3000
```

### LANCache Commands
```bash
# Container status
cd ~/lancache
docker compose ps

# View logs (all)
docker compose logs -f

# View specific container logs
docker logs -f lancache
docker logs -f lancache-dns

# Restart containers
docker compose restart

# Stop containers
docker compose down

# Start containers
docker compose up -d

# Check cache size
du -sh /cache/lancache-data

# Monitor cache growth
watch -n 5 'du -sh /cache/lancache-data'

# Test LANCache DNS
dig @127.0.0.1 -p 5380 steamcontent.com
```

### Firewall Commands
```bash
# View rules
sudo iptables -L -v -n
sudo iptables -t nat -L -v -n

# Save rules
sudo netfilter-persistent save

# Reload rules
sudo netfilter-persistent reload

# UFW status
sudo ufw status verbose
```

### Log Commands
```bash
# System log
sudo tail -f /var/log/syslog

# Service logs
sudo journalctl -u <service-name> -f

# All errors
sudo journalctl -p err -since today

# Docker logs
docker logs -f <container-name>

# Clean old logs
sudo journalctl --vacuum-time=30d
```

## Configuration File Locations

| Service | Configuration File |
|---------|-------------------|
| hostapd | `/etc/hostapd/hostapd.conf` |
| dnsmasq | `/etc/dnsmasq.conf` |
| netplan | `/etc/netplan/01-netcfg.yaml` |
| AdGuard | `/opt/AdGuardHome/AdGuardHome.yaml` |
| LANCache | `~/lancache/docker-compose.yml` |
| iptables | `/etc/iptables/rules.v4` |

## Important IP Addresses

| Device/Service | IP Address | Port |
|---------------|------------|------|
| Server Gateway | 192.168.1.1 | - |
| AdGuard Web UI | 192.168.1.1 | 3000 |
| AdGuard DNS | 192.168.1.1 | 53 |
| LANCache HTTP | 192.168.1.1 | 80, 443 |
| LANCache DNS | 127.0.0.1 | 5380 |
| Netdata | 192.168.1.1 | 19999 |
| SSH | 192.168.1.1 | 22 |

## DHCP Range

| Type | IP Range |
|------|----------|
| Static/Reserved | 192.168.1.2 - 192.168.1.49 |
| DHCP Pool | 192.168.1.50 - 192.168.1.250 |
| Reserved | 192.168.1.251 - 192.168.1.254 |

## Common Issues Quick Fixes

### No Internet
```bash
# Check modem
sudo mmcli -L && sudo mmcli -m 0

# Restart connection
sudo systemctl restart ModemManager
sudo nmcli connection up cellular

# Check routing
ip route show
```

### WiFi Not Working
```bash
# Check and restart
sudo systemctl status hostapd
sudo systemctl restart hostapd

# Check logs
sudo journalctl -u hostapd -n 50
```

### DNS Issues
```bash
# Check AdGuard
sudo systemctl status AdGuardHome
sudo systemctl restart AdGuardHome

# Test DNS
dig @192.168.1.1 google.com
```

### LANCache Not Working
```bash
# Check containers
cd ~/lancache
docker compose ps
docker compose logs

# Restart
docker compose restart
```

## Performance Monitoring

### Real-time Monitoring
```bash
# CPU and memory
htop

# Disk I/O
sudo iotop

# Network traffic
sudo iftop

# All-in-one dashboard
/usr/local/bin/dashboard.sh
```

### Check Metrics
```bash
# System load
uptime

# Disk space
df -h

# Memory usage
free -h

# Network stats
ip -s link

# Cellular signal
sudo mmcli -m 0 | grep signal

# WiFi clients
sudo iw dev wlan0 station dump | grep Station | wc -l

# Cache size
du -sh /cache/lancache-data
```

## Maintenance Commands

### Update System
```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Update Docker images
cd ~/lancache
docker compose pull
docker compose up -d

# Update firmware
sudo fwupdmgr refresh
sudo fwupdmgr update
```

### Backup
```bash
# Run backup script
/usr/local/bin/backup-config.sh

# Manual backup
sudo tar -czf ~/backup-$(date +%Y%m%d).tar.gz \
  /etc/hostapd/ \
  /etc/netplan/ \
  /etc/dnsmasq.conf \
  ~/lancache/ \
  /opt/AdGuardHome/
```

### Clean Up
```bash
# Clean apt cache
sudo apt autoclean
sudo apt autoremove

# Clean Docker
docker system prune -a

# Clean logs
sudo journalctl --vacuum-time=30d

# Clean old backups
find ~/backups -mtime +30 -delete
```

## Restart Services

### Individual Services
```bash
sudo systemctl restart ModemManager
sudo systemctl restart hostapd
sudo systemctl restart dnsmasq
sudo systemctl restart AdGuardHome
```

### All Network Services
```bash
sudo systemctl restart ModemManager hostapd dnsmasq AdGuardHome
cd ~/lancache && docker compose restart
```

### Full System Reboot
```bash
sudo reboot
```

## Troubleshooting Workflow

1. **Check service status**
   ```bash
   sudo systemctl status <service>
   ```

2. **View logs**
   ```bash
   sudo journalctl -u <service> -n 50
   ```

3. **Test connectivity**
   ```bash
   ping 8.8.8.8
   dig google.com
   ```

4. **Restart service**
   ```bash
   sudo systemctl restart <service>
   ```

5. **Check firewall**
   ```bash
   sudo iptables -L -v -n
   ```

6. **If all fails, reboot**
   ```bash
   sudo reboot
   ```

## Web Interfaces

| Service | URL | Default Credentials |
|---------|-----|-------------------|
| AdGuard Home | http://192.168.1.1:3000 | Set during install |
| Netdata | http://192.168.1.1:19999 | No auth by default |
| Grafana | http://192.168.1.1:3001 | admin/admin |

## Emergency Recovery

### Reset WiFi
```bash
sudo systemctl stop hostapd
sudo cp /etc/hostapd/hostapd.conf.bak /etc/hostapd/hostapd.conf
sudo systemctl start hostapd
```

### Reset Network
```bash
sudo systemctl stop hostapd dnsmasq
sudo netplan apply
sudo systemctl start dnsmasq hostapd
```

### Reset Modem
```bash
sudo mmcli -m 0 --reset
sudo systemctl restart ModemManager
sleep 30
sudo nmcli connection up cellular
```

### Full Network Reset
```bash
sudo systemctl stop hostapd dnsmasq ModemManager AdGuardHome
sudo netplan apply
sleep 5
sudo systemctl start ModemManager hostapd dnsmasq AdGuardHome
```

## Useful Aliases

Add to `~/.bashrc`:
```bash
# Service shortcuts
alias status='sudo systemctl status ModemManager hostapd dnsmasq AdGuardHome'
alias restart-network='sudo systemctl restart ModemManager hostapd dnsmasq'
alias check-signal='sudo mmcli -m 0 | grep signal'
alias wifi-clients='sudo iw dev wlan0 station dump | grep Station | wc -l'
alias cache-size='du -sh /cache/lancache-data'

# Log shortcuts
alias logs-modem='sudo journalctl -u ModemManager -f'
alias logs-wifi='sudo journalctl -u hostapd -f'
alias logs-dns='sudo journalctl -u AdGuardHome -f'
alias logs-cache='docker logs -f lancache'

# Monitoring shortcuts
alias monitor='htop'
alias network-monitor='sudo iftop'
alias disk-monitor='sudo iotop'

# Quick info
alias server-info='/usr/local/bin/server-status.sh'
```

Then reload:
```bash
source ~/.bashrc
```

## Security Commands

### Check SSH Sessions
```bash
who
w
last
```

### Check Failed Login Attempts
```bash
sudo grep "Failed password" /var/log/auth.log
```

### Update Passwords
```bash
# User password
passwd

# AdGuard password
# Use web interface
```

### Check Open Ports
```bash
sudo ss -tulpn | grep LISTEN
```

## Performance Tuning

### Check Current Settings
```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv4.tcp_congestion_control
```

### Apply Optimizations
```bash
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.wmem_max=134217728
```

### Make Persistent
```bash
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Testing Commands

### Speed Test
```bash
speedtest-cli
# or
curl -s https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py | python3 -
```

### DNS Test
```bash
# Test resolution
dig @192.168.1.1 google.com

# Test with timing
time nslookup google.com 192.168.1.1

# Test LANCache DNS
dig @127.0.0.1 -p 5380 steamcontent.com
```

### Network Test
```bash
# Local latency
ping -c 10 192.168.1.1

# Internet latency
ping -c 10 8.8.8.8

# Traceroute
traceroute google.com

# Bandwidth test (requires iperf3 on both ends)
# Server: iperf3 -s
# Client: iperf3 -c 192.168.1.1
```

### Cache Test
```bash
# Test HTTP
curl -I http://192.168.1.1

# Test with specific domain
curl -I http://steamcontent.com

# Download test file to populate cache
wget http://steamcontent.com/test.bin
```

## Quick Setup Commands

### First Time Setup (Summary)
```bash
# 1. Update system
sudo apt update && sudo apt upgrade -y

# 2. Install modem tools
sudo apt install -y modemmanager network-manager libqmi-utils

# 3. Install hostapd
sudo apt install -y hostapd

# 4. Install Docker
curl -fsSL https://get.docker.com | sh

# 5. Install AdGuard
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v

# 6. Install dnsmasq
sudo apt install -y dnsmasq

# 7. Install monitoring
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
```

## Documentation Links

- **Hardware Guide**: HARDWARE_GUIDE.md
- **Software Setup**: SOFTWARE_SETUP.md
- **Network Architecture**: NETWORK_ARCHITECTURE.md
- **Troubleshooting**: TROUBLESHOOTING.md
- **Advanced Config**: ADVANCED_CONFIGURATION.md
- **Quick Start**: QUICK_START.md

## Support Resources

- GitHub Issues: [Repository Issues](https://github.com/th3cavalry/Custom-wifi-AP/issues)
- Reddit: r/homelab, r/selfhosted
- Documentation: See individual service docs

---

**Tip**: Print this cheat sheet and keep it handy for quick reference!
