# Troubleshooting and FAQ Guide

## Common Issues and Solutions

### Modem Issues

#### Issue: Modem Not Detected

**Symptoms:**
- `mmcli -L` shows no modems
- `lsusb` or `lspci` doesn't show the device

**Solutions:**

1. **Check physical connection:**
   ```bash
   # Check PCIe devices
   lspci | grep -i modem
   lspci | grep -i wireless
   lspci | grep -i qualcomm
   
   # Check USB devices (if using USB adapter)
   lsusb | grep -i quectel
   ```

2. **Verify power:**
   - Ensure modem is receiving power
   - Some M.2 adapters require additional power connector

3. **Check BIOS:**
   - Enter BIOS and verify PCIe slot is enabled
   - Ensure M.2/PCIe device is recognized in BIOS

4. **Load kernel modules:**
   ```bash
   sudo modprobe qmi_wwan
   sudo modprobe cdc_mbim
   sudo modprobe option
   
   # Check if modules loaded
   lsmod | grep qmi
   ```

5. **Restart ModemManager:**
   ```bash
   sudo systemctl restart ModemManager
   sleep 10
   sudo mmcli -L
   ```

#### Issue: Low Signal Strength

**Symptoms:**
- Signal quality below 50%
- Slow or intermittent connection
- Frequent disconnections

**Solutions:**

1. **Check signal:**
   ```bash
   sudo mmcli -m 0 | grep signal
   # Look for signal quality percentage
   ```

2. **Optimize antenna placement:**
   - Position antennas vertically
   - Keep antennas away from metal objects
   - Try different locations in your home
   - Consider outdoor antenna mounting

3. **Check antenna connections:**
   - Verify all 4 antennas are connected
   - Ensure connectors are tight
   - Check for damaged cables

4. **Find best carrier:**
   ```bash
   # Scan for networks
   sudo mmcli -m 0 --scan
   
   # Manually select carrier if needed
   sudo mmcli -m 0 --set-operator-id="your-carrier-id"
   ```

5. **Check band lock:**
   ```bash
   # Lock to best performing band (advanced)
   # Requires AT commands via QMI
   # See Quectel documentation
   ```

#### Issue: Connection Drops Frequently

**Symptoms:**
- Internet works then stops
- Have to manually restart connection
- "Connection lost" errors

**Solutions:**

1. **Enable connection watchdog:**
   
   Create `/usr/local/bin/modem-watchdog.sh`:
   ```bash
   #!/bin/bash
   INTERFACE="wwan0"
   CONNECTION="cellular"
   PING_HOST="8.8.8.8"
   
   while true; do
       if ! ping -c 3 -W 5 -I $INTERFACE $PING_HOST > /dev/null 2>&1; then
           echo "$(date): Connection down, restarting..."
           nmcli connection down $CONNECTION
           sleep 5
           nmcli connection up $CONNECTION
           sleep 30
       fi
       sleep 60
   done
   ```
   
   Make executable and create systemd service:
   ```bash
   sudo chmod +x /usr/local/bin/modem-watchdog.sh
   
   # Create service
   sudo nano /etc/systemd/system/modem-watchdog.service
   ```
   
   Add:
   ```ini
   [Unit]
   Description=Modem Connection Watchdog
   After=network-online.target ModemManager.service
   Wants=network-online.target
   
   [Service]
   Type=simple
   ExecStart=/usr/local/bin/modem-watchdog.sh
   Restart=always
   RestartSec=10
   
   [Install]
   WantedBy=multi-user.target
   ```
   
   Enable:
   ```bash
   sudo systemctl enable modem-watchdog
   sudo systemctl start modem-watchdog
   ```

2. **Check data usage:**
   - Verify you haven't exceeded carrier limits
   - Check for throttling

3. **Update modem firmware:**
   - Visit Quectel website for latest firmware
   - Follow firmware update procedures

4. **Check for overheating:**
   ```bash
   # Monitor temperature
   sudo mmcli -m 0 | grep temperature
   ```
   - Add heatsink if needed
   - Improve case airflow

#### Issue: Slow Speeds

**Symptoms:**
- Speed tests show slower than expected
- High latency
- Buffering during streaming

**Solutions:**

1. **Check connection type:**
   ```bash
   sudo mmcli -m 0 | grep "access tech"
   # Should show 5G or LTE
   ```

2. **Verify APN settings:**
   ```bash
   nmcli connection show cellular | grep apn
   # Ensure correct APN for your carrier
   ```

3. **Check network congestion:**
   - Test at different times of day
   - Check carrier coverage map

4. **Optimize TCP settings:**
   Add to `/etc/sysctl.conf`:
   ```ini
   net.core.rmem_max=134217728
   net.core.wmem_max=134217728
   net.ipv4.tcp_rmem=4096 87380 67108864
   net.ipv4.tcp_wmem=4096 65536 67108864
   net.ipv4.tcp_congestion_control=bbr
   ```
   
   Apply:
   ```bash
   sudo sysctl -p
   ```

5. **Disable IPv6 (if causing issues):**
   ```bash
   sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
   echo "net.ipv6.conf.all.disable_ipv6=1" | sudo tee -a /etc/sysctl.conf
   ```

### WiFi Issues

#### Issue: hostapd Won't Start

**Symptoms:**
- `systemctl status hostapd` shows failed
- WiFi network not visible
- Error in logs

**Solutions:**

1. **Check logs:**
   ```bash
   sudo journalctl -u hostapd -n 50 --no-pager
   ```

2. **Verify interface:**
   ```bash
   # Check if WiFi interface exists
   ip link show wlan0
   
   # Check if interface is up
   sudo ip link set wlan0 up
   ```

3. **Check for conflicts:**
   ```bash
   # Make sure NetworkManager isn't managing WiFi
   sudo nmcli device set wlan0 managed no
   ```

4. **Test minimal config:**
   
   Backup current config:
   ```bash
   sudo cp /etc/hostapd/hostapd.conf /etc/hostapd/hostapd.conf.bak
   ```
   
   Create minimal test config:
   ```bash
   sudo nano /etc/hostapd/hostapd.conf
   ```
   
   ```ini
   interface=wlan0
   driver=nl80211
   ssid=TestNetwork
   hw_mode=g
   channel=6
   ```
   
   Try starting:
   ```bash
   sudo systemctl restart hostapd
   ```
   
   If works, gradually add features back.

5. **Check kernel modules:**
   ```bash
   # Check if WiFi driver loaded
   lsmod | grep -i wifi
   lsmod | grep 80211
   
   # Reload modules if needed
   sudo modprobe -r iwlwifi  # Example for Intel
   sudo modprobe iwlwifi
   ```

#### Issue: WiFi 7 Features Not Working

**Symptoms:**
- WiFi works but not at WiFi 7 speeds
- 160MHz channels not available
- 6GHz band not working

**Solutions:**

1. **Check kernel version:**
   ```bash
   uname -r
   # Need kernel 6.2+ for WiFi 7
   ```
   
   Update if needed:
   ```bash
   sudo apt update
   sudo apt install --install-recommends linux-generic-hwe-24.04
   sudo reboot
   ```

2. **Verify regulatory domain:**
   ```bash
   iw reg get
   ```
   
   Set correct country code:
   ```bash
   sudo iw reg set US  # Replace with your country
   ```
   
   Make persistent in `/etc/hostapd/hostapd.conf`:
   ```ini
   country_code=US
   ```

3. **Check available channels:**
   ```bash
   iw list | grep -A 10 "Frequencies"
   ```

4. **Verify client compatibility:**
   - Ensure client devices support WiFi 7
   - Check client can use 160MHz channels
   - Verify 6GHz band support on client

5. **Update WiFi firmware:**
   ```bash
   sudo apt install linux-firmware
   sudo update-initramfs -u
   sudo reboot
   ```

#### Issue: WiFi Clients Can't Connect

**Symptoms:**
- Network visible but connection fails
- Wrong password errors
- Authentication timeout

**Solutions:**

1. **Verify password:**
   - Check `/etc/hostapd/hostapd.conf`
   - Ensure password meets requirements (8+ characters)

2. **Check WPA3 compatibility:**
   - Some older devices don't support WPA3
   - Try WPA2/WPA3 mixed mode:
   ```ini
   wpa=2
   wpa_key_mgmt=WPA-PSK SAE
   rsn_pairwise=CCMP
   ```

3. **Verify DHCP:**
   ```bash
   sudo systemctl status dnsmasq
   # Check DHCP range configured
   ```

4. **Check iptables:**
   ```bash
   sudo iptables -L -v -n
   # Ensure FORWARD rules allow traffic
   ```

5. **Test with open network:**
   Temporarily remove encryption:
   ```ini
   # Comment out in hostapd.conf
   #wpa=2
   #wpa_key_mgmt=SAE
   #wpa_passphrase=...
   ```
   
   If works, issue is with encryption/auth.

#### Issue: Slow WiFi Speeds

**Symptoms:**
- WiFi works but slower than expected
- Speed tests show low throughput
- Lag during gaming or streaming

**Solutions:**

1. **Check channel congestion:**
   ```bash
   sudo iw dev wlan0 scan | grep -E "SSID|freq|signal"
   ```
   
   Change to less congested channel in hostapd.conf.

2. **Verify 5GHz is being used:**
   ```bash
   iw dev wlan0 info
   # Check frequency
   ```

3. **Check client connection details:**
   ```bash
   sudo iw dev wlan0 station dump
   # Look for tx/rx rates
   ```

4. **Optimize hostapd settings:**
   ```ini
   # Add to hostapd.conf
   wmm_enabled=1
   he_su_beamformer=1
   he_su_beamformee=1
   he_mu_beamformer=1
   ```

5. **Check for interference:**
   - Move away from microwaves, Bluetooth devices
   - Try different antenna orientation
   - Use 5GHz band when possible

### AdGuard Home Issues

#### Issue: DNS Not Resolving

**Symptoms:**
- Websites won't load
- "DNS_PROBE_FINISHED_NXDOMAIN" errors
- Can ping IPs but not domain names

**Solutions:**

1. **Check AdGuard service:**
   ```bash
   sudo systemctl status AdGuardHome
   ```

2. **Verify DNS port:**
   ```bash
   sudo ss -tulpn | grep :53
   # Should show AdGuard listening
   ```

3. **Test DNS directly:**
   ```bash
   dig @192.168.1.1 google.com
   nslookup google.com 192.168.1.1
   ```

4. **Check upstream DNS:**
   - Log into AdGuard web UI
   - Settings → DNS settings
   - Test upstream DNS servers

5. **Clear DNS cache:**
   ```bash
   sudo systemctl restart AdGuardHome
   ```

#### Issue: Ads Not Being Blocked

**Symptoms:**
- Still seeing advertisements
- AdGuard shows queries but no blocks
- Block rate is 0%

**Solutions:**

1. **Verify blocklists are enabled:**
   - Web UI → Filters → DNS blocklists
   - Ensure lists are checked and updated

2. **Force blocklist update:**
   - Click "Update" on each list
   - Wait for completion

3. **Check if domains are whitelisted:**
   - Settings → Filters → Custom filtering rules
   - Remove any whitelist entries for ad domains

4. **Test with known ad domain:**
   ```bash
   dig @192.168.1.1 ads.google.com
   # Should return blocked address
   ```

5. **Verify clients using correct DNS:**
   ```bash
   # On client device:
   nslookup google.com
   # Should show 192.168.1.1 as server
   ```

#### Issue: LANCache DNS Not Working

**Symptoms:**
- Game downloads not being cached
- LANCache hit rate is 0%
- DNS not redirecting game domains

**Solutions:**

1. **Check LANCache DNS container:**
   ```bash
   docker ps | grep lancache-dns
   docker logs lancache-dns
   ```

2. **Test LANCache DNS:**
   ```bash
   dig @127.0.0.1 -p 5380 steamcontent.com
   # Should return 192.168.1.1
   ```

3. **Verify AdGuard DNS rewrites:**
   - Web UI → Settings → DNS settings
   - Scroll to "Upstream DNS servers"
   - Ensure conditional forwarding rules exist

4. **Check Docker network:**
   ```bash
   docker network ls
   docker network inspect lancache_default
   ```

5. **Restart both services:**
   ```bash
   cd ~/lancache
   docker compose restart
   sudo systemctl restart AdGuardHome
   ```

### LANCache Issues

#### Issue: Cache Not Working

**Symptoms:**
- Game downloads still slow
- Not seeing cache hits
- Docker logs show errors

**Solutions:**

1. **Check container status:**
   ```bash
   docker ps
   docker compose ps
   ```

2. **View logs:**
   ```bash
   cd ~/lancache
   docker compose logs -f lancache
   ```

3. **Test HTTP endpoint:**
   ```bash
   curl -I http://192.168.1.1
   # Should return LANCache headers
   ```

4. **Check cache disk:**
   ```bash
   df -h /cache
   # Ensure space available
   
   du -sh /cache/lancache-data
   # Check cache size
   ```

5. **Verify ports:**
   ```bash
   sudo ss -tulpn | grep -E ':80|:443'
   # Should show Docker containers
   ```

6. **Restart containers:**
   ```bash
   cd ~/lancache
   docker compose down
   docker compose up -d
   ```

#### Issue: Cache Disk Full

**Symptoms:**
- Cache not accepting new content
- Disk space at 100%
- Container errors about disk space

**Solutions:**

1. **Check disk usage:**
   ```bash
   df -h /cache
   du -sh /cache/lancache-data/*
   ```

2. **Clean old cache:**
   ```bash
   # Stop containers
   cd ~/lancache
   docker compose down
   
   # Remove old cache (careful!)
   # This will force re-download
   sudo rm -rf /cache/lancache-data/*
   
   # Start containers
   docker compose up -d
   ```

3. **Adjust cache size:**
   Edit `docker-compose.yml`:
   ```yaml
   environment:
     - CACHE_DISK_SIZE=3500g  # Reduce this
   ```
   
   ```bash
   docker compose down
   docker compose up -d
   ```

4. **Add more storage:**
   - Mount additional drive
   - Update Docker volumes

#### Issue: Low Cache Hit Rate

**Symptoms:**
- Cache exists but downloads still slow
- Hit rate under 20%
- Not seeing benefit

**Solutions:**

1. **Check DNS routing:**
   ```bash
   # From client, verify DNS resolution
   nslookup steamcontent.com
   # Should return 192.168.1.1
   ```

2. **Verify game launcher settings:**
   - Steam: Should use default CDN
   - Epic: Ensure not using VPN
   - Check launcher isn't bypassing DNS

3. **Monitor cache in real-time:**
   ```bash
   # Watch cache access logs
   docker logs -f lancache
   
   # Monitor cache size growth
   watch -n 2 'du -sh /cache/lancache-data'
   ```

4. **Warm up cache:**
   - Download popular games on one PC
   - Subsequent downloads on other PCs will be cached

5. **Check supported services:**
   ```bash
   docker exec lancache cat /etc/nginx/conf.d/lancache.conf
   # Verify game services are configured
   ```

### Network Performance Issues

#### Issue: High Latency

**Symptoms:**
- Ping times over 100ms
- Lag in online games
- Delayed responses

**Solutions:**

1. **Check cellular latency:**
   ```bash
   ping -I wwan0 8.8.8.8
   # Test WAN latency
   
   ping 192.168.1.1
   # Test local latency (should be <5ms)
   ```

2. **Check WiFi latency:**
   ```bash
   # From WiFi client
   ping 192.168.1.1
   # Should be 1-5ms
   ```

3. **QoS configuration:**
   Implement traffic prioritization (see NETWORK_ARCHITECTURE.md)

4. **Check system load:**
   ```bash
   htop
   # Ensure CPU not overloaded
   
   sudo iotop
   # Check for disk I/O issues
   ```

5. **Optimize network stack:**
   ```bash
   # Check current settings
   sysctl net.ipv4.tcp_congestion_control
   
   # Enable BBR
   echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
   sudo sysctl -p
   ```

#### Issue: Intermittent Packet Loss

**Symptoms:**
- Dropped connections
- Stuttering in games/video
- Ping shows packet loss

**Solutions:**

1. **Test packet loss:**
   ```bash
   ping -c 100 8.8.8.8
   # Look for packet loss percentage
   ```

2. **Check WiFi signal:**
   ```bash
   sudo iw dev wlan0 station dump
   # Check signal strength for each client
   ```

3. **Monitor errors:**
   ```bash
   ip -s link show wlan0
   ip -s link show wwan0
   # Look for errors/drops
   ```

4. **Check for interference:**
   - Run during different times
   - Change WiFi channel
   - Move antennas

5. **Verify MTU settings:**
   ```bash
   ip link show wwan0
   # Check MTU size
   
   # Adjust if needed
   sudo ip link set wwan0 mtu 1500
   ```

## Frequently Asked Questions

### General Questions

**Q: How much will this cost total?**  
A: Approximately $600-1,100 for hardware additions, plus ~$50-100/month for cellular data plan.

**Q: What cellular carriers work best?**  
A: T-Mobile and Verizon generally have best 5G coverage. Check coverage at your address.

**Q: Can I use this with existing internet?**  
A: Yes! Use ethernet instead of cellular modem for WAN connection.

**Q: Will this work in rural areas?**  
A: Yes, if you have 4G/5G coverage. Check carrier coverage maps first.

**Q: How much power does this use?**  
A: Approximately 200W idle, 350W under load. ~$30-40/month electricity at average US rates.

### Hardware Questions

**Q: Can I use a different PC?**  
A: Any system with PCIe slots will work. Adjust guides accordingly.

**Q: Do I need the exact hardware listed?**  
A: No, but verify compatibility. Alternatives exist for most components.

**Q: Can I use WiFi 6E instead of WiFi 7?**  
A: Yes! Intel AX210 is good alternative (~$30). Adjust hostapd config.

**Q: What if I already have a modem?**  
A: Most USB modems will work. Adjust network setup accordingly.

**Q: Can I add more storage later?**  
A: Yes, T7810 has multiple drive bays and SATA ports.

### Software Questions

**Q: Can I use Windows instead of Linux?**  
A: Possible but not recommended. Many tools are Linux-specific.

**Q: Do I need to know Linux?**  
A: Basic command line knowledge helpful. Guides are detailed.

**Q: Can I run other services too?**  
A: Yes! Consider Proxmox for running multiple VMs/containers.

**Q: How do I backup my configuration?**  
A: See backup scripts in SOFTWARE_SETUP.md.

**Q: Can I access the server remotely?**  
A: Yes, via SSH. Consider VPN for secure remote access.

### Performance Questions

**Q: What speeds can I expect?**  
A: Cellular: 100-1000 Mbps down (varies). WiFi 7: 1000-3000 Mbps (device dependent).

**Q: How many devices can connect?**  
A: Reliably 50-100 devices, depending on usage patterns.

**Q: Will 4K streaming work?**  
A: Yes, if cellular speeds are 50+ Mbps per stream.

**Q: What about online gaming latency?**  
A: 5G latency typically 20-50ms. Good for most gaming.

**Q: How much bandwidth does LANCache save?**  
A: 50-70% reduction for game downloads in multi-PC households.

### Troubleshooting Questions

**Q: What if modem isn't detected?**  
A: See "Modem Not Detected" section above.

**Q: WiFi network not showing up?**  
A: Check hostapd service and logs. See "WiFi Issues" section.

**Q: How do I reset everything?**  
A: Stop all services, backup configs, and reinstall Ubuntu if needed.

**Q: Where are the logs?**  
A: `/var/log/syslog`, `journalctl -u <service>`, and `docker logs`.

**Q: Can I get help with my specific issue?**  
A: Post in r/homelab or r/selfhosted with detailed description.

## Getting More Help

### Log Collection for Support

When asking for help, collect these logs:

```bash
# Create log collection script
cat > /tmp/collect-logs.sh << 'EOF'
#!/bin/bash
OUTPUT="/tmp/server-logs-$(date +%Y%m%d-%H%M%S).tar.gz"
TEMP_DIR="/tmp/log-collection"

mkdir -p $TEMP_DIR

# System info
uname -a > $TEMP_DIR/system-info.txt
lspci > $TEMP_DIR/lspci.txt
lsusb > $TEMP_DIR/lsusb.txt
ip addr > $TEMP_DIR/ip-addr.txt
ip route > $TEMP_DIR/ip-route.txt

# Service status
systemctl status ModemManager > $TEMP_DIR/modemmanager-status.txt 2>&1
systemctl status hostapd > $TEMP_DIR/hostapd-status.txt 2>&1
systemctl status dnsmasq > $TEMP_DIR/dnsmasq-status.txt 2>&1
systemctl status AdGuardHome > $TEMP_DIR/adguard-status.txt 2>&1

# Service logs
journalctl -u ModemManager -n 100 > $TEMP_DIR/modemmanager.log 2>&1
journalctl -u hostapd -n 100 > $TEMP_DIR/hostapd.log 2>&1
journalctl -u dnsmasq -n 100 > $TEMP_DIR/dnsmasq.log 2>&1

# Docker
docker ps > $TEMP_DIR/docker-ps.txt 2>&1
docker compose ps > $TEMP_DIR/docker-compose-ps.txt 2>&1
docker logs lancache --tail 100 > $TEMP_DIR/lancache.log 2>&1
docker logs lancache-dns --tail 100 > $TEMP_DIR/lancache-dns.log 2>&1

# Configs (sanitized)
grep -v "passphrase" /etc/hostapd/hostapd.conf > $TEMP_DIR/hostapd.conf 2>&1
cat /etc/dnsmasq.conf > $TEMP_DIR/dnsmasq.conf 2>&1
cat /etc/netplan/*.yaml > $TEMP_DIR/netplan.yaml 2>&1

# Create archive
tar -czf $OUTPUT -C /tmp log-collection
rm -rf $TEMP_DIR

echo "Logs collected: $OUTPUT"
echo "Review before sharing - contains system info!"
EOF

chmod +x /tmp/collect-logs.sh
/tmp/collect-logs.sh
```

### Community Resources

- **Reddit**: r/homelab, r/selfhosted, r/HomeNetworking
- **Discord**: LANCache Discord server
- **Forums**: Level1Techs, ServeTheHome
- **Stack Exchange**: Server Fault, Unix & Linux

### Professional Support

For complex issues or custom requirements:
- Hire a network consultant
- Contact component manufacturers
- Consider managed hosting services

## Preventive Maintenance

### Weekly Checks
```bash
# Run status check
/usr/local/bin/server-status.sh

# Check disk space
df -h

# Review logs for errors
sudo journalctl -p err -since yesterday
```

### Monthly Tasks
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Update Docker images
cd ~/lancache
docker compose pull
docker compose up -d

# Check and rotate logs
sudo journalctl --vacuum-time=30d

# Review AdGuard stats
# Check blocked percentage, top clients, etc.
```

### Quarterly Tasks
```bash
# Update firmware
sudo fwupdmgr refresh
sudo fwupdmgr update

# Full backup
/usr/local/bin/backup-config.sh

# Test restore procedure

# Review and optimize rules

# Check hardware health
sudo smartctl -a /dev/nvme0n1
```

## Performance Tuning Checklist

- [ ] Cellular signal optimized (>50% strength)
- [ ] WiFi channels selected (least congested)
- [ ] TCP BBR enabled
- [ ] DNS cache working (AdGuard)
- [ ] LANCache hit rate >60%
- [ ] Monitoring configured (Netdata)
- [ ] Backups automated
- [ ] Logs rotating properly
- [ ] Firewall rules verified
- [ ] QoS configured (if needed)

## When to Seek Professional Help

Consider professional assistance if:
- Cellular signal consistently poor (<30%)
- Hardware not being detected after troubleshooting
- Network security concerns
- Need enterprise-grade features
- Time constraints for DIY
- Complex custom requirements

---

Remember: Most issues are resolved by:
1. Checking logs
2. Verifying configurations
3. Restarting services
4. Ensuring latest updates

**Don't hesitate to ask for help in the community!**
