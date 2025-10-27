# Advanced Configuration and Optimization

## Advanced Networking

### VLAN Configuration

Segment your network for better security and performance.

#### Setup VLANs

1. **Install VLAN support:**
   ```bash
   sudo apt install -y vlan
   sudo modprobe 8021q
   echo "8021q" | sudo tee -a /etc/modules
   ```

2. **Configure VLANs in netplan:**

   Edit `/etc/netplan/01-netcfg.yaml`:
   ```yaml
   network:
     version: 2
     renderer: networkd
     
     ethernets:
       wwan0:
         dhcp4: yes
       
       eth0:
         dhcp4: no
     
     vlans:
       vlan10:
         id: 10
         link: eth0
         addresses: [192.168.10.1/24]
         
       vlan20:
         id: 20
         link: eth0
         addresses: [192.168.20.1/24]
         
       vlan30:
         id: 30
         link: eth0
         addresses: [192.168.30.1/24]
     
     wifis:
       wlan0:
         dhcp4: no
         addresses: [192.168.1.1/24]
   ```

3. **Configure DHCP per VLAN:**

   Edit `/etc/dnsmasq.conf`:
   ```ini
   # Main WiFi
   interface=wlan0
   dhcp-range=set:wlan,192.168.1.50,192.168.1.250,24h
   dhcp-option=tag:wlan,option:router,192.168.1.1
   
   # VLAN 10 - Main LAN
   interface=vlan10
   dhcp-range=set:vlan10,192.168.10.50,192.168.10.250,24h
   dhcp-option=tag:vlan10,option:router,192.168.10.1
   
   # VLAN 20 - Guest
   interface=vlan20
   dhcp-range=set:vlan20,192.168.20.50,192.168.20.250,4h
   dhcp-option=tag:vlan20,option:router,192.168.20.1
   
   # VLAN 30 - IoT
   interface=vlan30
   dhcp-range=set:vlan30,192.168.30.50,192.168.30.250,24h
   dhcp-option=tag:vlan30,option:router,192.168.30.1
   ```

4. **Configure firewall rules per VLAN:**
   ```bash
   # Allow all VLANs to internet
   sudo iptables -t nat -A POSTROUTING -o wwan0 -j MASQUERADE
   
   # Main LAN - full access
   sudo iptables -A FORWARD -i vlan10 -o wwan0 -j ACCEPT
   sudo iptables -A FORWARD -i wwan0 -o vlan10 -m state --state RELATED,ESTABLISHED -j ACCEPT
   
   # Guest - internet only, no LAN access
   sudo iptables -A FORWARD -i vlan20 -o wwan0 -j ACCEPT
   sudo iptables -A FORWARD -i wwan0 -o vlan20 -m state --state RELATED,ESTABLISHED -j ACCEPT
   sudo iptables -A FORWARD -i vlan20 -o vlan10 -j DROP
   sudo iptables -A FORWARD -i vlan20 -o vlan30 -j DROP
   
   # IoT - limited access
   sudo iptables -A FORWARD -i vlan30 -o wwan0 -j ACCEPT
   sudo iptables -A FORWARD -i wwan0 -o vlan30 -m state --state RELATED,ESTABLISHED -j ACCEPT
   sudo iptables -A FORWARD -i vlan30 -o vlan10 -j DROP
   
   # Save rules
   sudo netfilter-persistent save
   ```

### Multiple SSIDs with Different VLANs

Create separate WiFi networks for different purposes.

#### Using hostapd with VLANs

1. **Install hostapd with VLAN support:**
   ```bash
   sudo apt install -y hostapd
   ```

2. **Configure hostapd with multiple SSIDs:**

   Edit `/etc/hostapd/hostapd.conf`:
   ```ini
   interface=wlan0
   driver=nl80211
   
   # Primary SSID (Main Network)
   ssid=HomeNetwork
   wpa_passphrase=YourPassword123
   
   # VLAN support
   dynamic_vlan=1
   vlan_file=/etc/hostapd/hostapd.vlan
   
   # Security
   wpa=2
   wpa_key_mgmt=SAE
   rsn_pairwise=CCMP
   
   # WiFi 7 settings
   hw_mode=a
   channel=36
   ieee80211ac=1
   ieee80211ax=1
   ieee80211be=1
   country_code=US
   
   # Additional SSIDs
   bss=wlan0_1
   ssid=GuestNetwork
   wpa_passphrase=GuestPass123
   
   bss=wlan0_2
   ssid=IoTNetwork
   wpa_passphrase=IoTPass123
   ```

3. **Create VLAN mapping file:**

   Create `/etc/hostapd/hostapd.vlan`:
   ```
   # VLAN ID     Interface prefix
   1             wlan0
   10            wlan0.10
   20            wlan0.20
   30            wlan0.30
   ```

### QoS (Quality of Service) Configuration

Prioritize important traffic for better performance.

#### Using tc (Traffic Control)

Create `/usr/local/bin/setup-qos.sh`:

```bash
#!/bin/bash

# Configuration
WAN_INTERFACE="wwan0"
LAN_INTERFACE="wlan0"
WAN_BANDWIDTH="100mbit"  # Adjust to your connection speed

# Clear existing rules
tc qdisc del dev $WAN_INTERFACE root 2>/dev/null
tc qdisc del dev $LAN_INTERFACE root 2>/dev/null

# Setup root qdisc
tc qdisc add dev $WAN_INTERFACE root handle 1: htb default 30

# Create main class
tc class add dev $WAN_INTERFACE parent 1: classid 1:1 htb rate $WAN_BANDWIDTH ceil $WAN_BANDWIDTH

# High priority (gaming, VoIP)
tc class add dev $WAN_INTERFACE parent 1:1 classid 1:10 htb rate 50mbit ceil $WAN_BANDWIDTH prio 1
tc qdisc add dev $WAN_INTERFACE parent 1:10 handle 10: sfq perturb 10

# Medium priority (streaming, web)
tc class add dev $WAN_INTERFACE parent 1:1 classid 1:20 htb rate 30mbit ceil 80mbit prio 2
tc qdisc add dev $WAN_INTERFACE parent 1:20 handle 20: sfq perturb 10

# Low priority (downloads, updates)
tc class add dev $WAN_INTERFACE parent 1:1 classid 1:30 htb rate 20mbit ceil 60mbit prio 3
tc qdisc add dev $WAN_INTERFACE parent 1:30 handle 30: sfq perturb 10

# Filters for high priority
# Gaming ports
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 1 u32 match ip dport 27015 0xffff flowid 1:10  # Source
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 1 u32 match ip dport 3074 0xffff flowid 1:10   # Xbox Live
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 1 u32 match ip dport 3478 0xffff flowid 1:10   # PSN
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 1 u32 match ip dport 25565 0xffff flowid 1:10  # Minecraft

# VoIP ports
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 1 u32 match ip dport 5060 0xffff flowid 1:10   # SIP
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 1 u32 match ip sport 50000 0xffff flowid 1:10  # Discord

# DNS (high priority)
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 1 u32 match ip dport 53 0xffff flowid 1:10
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 1 u32 match ip sport 53 0xffff flowid 1:10

# ICMP (ping)
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 1 u32 match ip protocol 1 0xff flowid 1:10

# Filters for medium priority (HTTP/HTTPS)
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 2 u32 match ip dport 80 0xffff flowid 1:20
tc filter add dev $WAN_INTERFACE protocol ip parent 1:0 prio 2 u32 match ip dport 443 0xffff flowid 1:20

# Everything else goes to low priority (default 30)

echo "QoS configured on $WAN_INTERFACE"
echo "Bandwidth limit: $WAN_BANDWIDTH"
```

Make executable and run:
```bash
sudo chmod +x /usr/local/bin/setup-qos.sh
sudo /usr/local/bin/setup-qos.sh
```

Create systemd service to run on boot:
```bash
sudo nano /etc/systemd/system/qos.service
```

```ini
[Unit]
Description=Setup QoS
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/setup-qos.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable qos
sudo systemctl start qos
```

### Load Balancing (Multiple WAN)

If you have multiple internet connections.

#### Using iproute2 for load balancing

1. **Configure routing tables:**

   Edit `/etc/iproute2/rt_tables`:
   ```
   100 cellular
   101 ethernet
   ```

2. **Setup routing rules:**

   Create `/usr/local/bin/setup-loadbalance.sh`:
   ```bash
   #!/bin/bash
   
   # WAN interfaces
   WAN1="wwan0"
   WAN1_GW="192.168.100.1"  # Cellular gateway
   WAN2="eth1"
   WAN2_GW="192.168.200.1"  # Ethernet gateway
   
   # Routing tables
   ip route add default via $WAN1_GW dev $WAN1 table cellular
   ip route add default via $WAN2_GW dev $WAN2 table ethernet
   
   # Rules
   ip rule add from 192.168.1.0/24 table cellular prio 100
   ip rule add from 192.168.10.0/24 table ethernet prio 101
   
   # Default load balancing
   ip route add default scope global \
     nexthop via $WAN1_GW dev $WAN1 weight 1 \
     nexthop via $WAN2_GW dev $WAN2 weight 1
   ```

## Advanced WiFi Configuration

### WiFi 7 Maximum Performance

Optimize for absolute maximum WiFi performance.

```ini
# /etc/hostapd/hostapd.conf

interface=wlan0
driver=nl80211
ssid=UltraFastNetwork

# WiFi 7 on 6GHz (if supported)
hw_mode=a
op_class=131
channel=15
he_oper_chwidth=2  # 160MHz
ieee80211ac=1
ieee80211ax=1
ieee80211be=1

# Maximum performance settings
wmm_enabled=1
he_su_beamformer=1
he_su_beamformee=1
he_mu_beamformer=1
eht_su_beamformer=1
eht_su_beamformee=1
eht_mu_beamformer=1

# BSS Color for better interference handling
he_bss_color=3

# Multi-link operation (MLO) for WiFi 7
#eht_enable_mlo=1

# Security
wpa=2
wpa_key_mgmt=SAE
wpa_passphrase=YourSecurePassword
rsn_pairwise=CCMP
group_cipher=CCMP

# Country and regulatory
country_code=US
ieee80211d=1
ieee80211h=1

# Performance tuning
beacon_int=100
dtim_period=2
max_num_sta=100
rts_threshold=2347
fragm_threshold=2346
```

### Mesh Networking

Create a mesh network with multiple access points.

#### Using batman-adv

1. **Install batman-adv:**
   ```bash
   sudo apt install -y batctl alfred
   ```

2. **Configure mesh:**
   ```bash
   # Load module
   sudo modprobe batman-adv
   
   # Configure interface
   sudo batctl if add wlan0
   sudo ifconfig bat0 up
   sudo ifconfig bat0 192.168.50.1/24
   ```

3. **Configure mesh bridge:**
   ```bash
   sudo brctl addbr br-mesh
   sudo brctl addif br-mesh bat0 wlan0
   sudo ifconfig br-mesh up
   ```

## Advanced LANCache Configuration

### Cache Optimization

#### Tune Nginx for LANCache

Create custom Nginx configuration:

```bash
docker exec -it lancache bash
cd /etc/nginx
```

Edit `nginx.conf` for performance:
```nginx
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 1000;
    
    # Cache settings
    proxy_cache_path /data/cache 
        levels=2:2 
        keys_zone=cache:500m 
        max_size=3500g 
        inactive=3650d 
        use_temp_path=off;
    
    # Buffer sizes
    proxy_buffer_size 128k;
    proxy_buffers 8 128k;
    proxy_busy_buffers_size 256k;
    
    # Timeouts
    proxy_connect_timeout 300;
    proxy_send_timeout 300;
    proxy_read_timeout 300;
}
```

#### Cache Warmup Script

Create `/usr/local/bin/lancache-warmup.sh`:

```bash
#!/bin/bash

# List of popular game manifests to pre-cache
URLS=(
    "https://steamcdn-a.akamaihd.net/steam/apps/730/manifest.json"  # CS2
    "https://epicgames-download1.akamaihd.net/Builds/Fortnite/"
    # Add more as needed
)

for url in "${URLS[@]}"; do
    echo "Pre-caching: $url"
    curl -s -o /dev/null "$url"
done

echo "Cache warmup complete"
```

### LANCache Monitoring Dashboard

#### Using Prometheus and Grafana

1. **Add Prometheus monitoring to LANCache:**

   Update `docker-compose.yml`:
   ```yaml
   services:
     # ... existing services ...
     
     prometheus:
       image: prom/prometheus:latest
       container_name: prometheus
       volumes:
         - ./prometheus.yml:/etc/prometheus/prometheus.yml
         - prometheus-data:/prometheus
       ports:
         - "9090:9090"
       restart: unless-stopped
     
     grafana:
       image: grafana/grafana:latest
       container_name: grafana
       volumes:
         - grafana-data:/var/lib/grafana
       ports:
         - "3001:3000"
       restart: unless-stopped
       environment:
         - GF_SECURITY_ADMIN_PASSWORD=admin
   
   volumes:
     prometheus-data:
     grafana-data:
   ```

2. **Configure Prometheus:**

   Create `prometheus.yml`:
   ```yaml
   global:
     scrape_interval: 15s
   
   scrape_configs:
     - job_name: 'lancache'
       static_configs:
         - targets: ['lancache:80']
     
     - job_name: 'node'
       static_configs:
         - targets: ['host.docker.internal:9100']
   ```

3. **Install node exporter on host:**
   ```bash
   sudo apt install -y prometheus-node-exporter
   sudo systemctl enable prometheus-node-exporter
   sudo systemctl start prometheus-node-exporter
   ```

## Advanced AdGuard Configuration

### Custom DNS Filtering Rules

#### Block Tracking and Telemetry

Add to AdGuard → Filters → Custom filtering rules:

```
# Microsoft telemetry
||telemetry.microsoft.com^
||vortex.data.microsoft.com^
||vortex-win.data.microsoft.com^

# Google analytics
||google-analytics.com^
||googletagmanager.com^
||doubleclick.net^

# Social media trackers
||facebook.com/tr^
||connect.facebook.net^
||facebook.com/plugins^

# Amazon tracking
||amazon-adsystem.com^

# Gaming telemetry
||tracking.epicgames.com^
||metrics.origin.com^
```

### DNS-over-HTTPS (DoH) Upstream

Configure encrypted upstream DNS:

```yaml
# In AdGuard settings
upstream_dns:
  - https://dns.cloudflare.com/dns-query
  - https://dns.google/dns-query
  - https://doh.opendns.com/dns-query
  - tls://1.1.1.1
  - tls://1.0.0.1

bootstrap_dns:
  - 1.1.1.1
  - 8.8.8.8
```

### Split DNS Configuration

Route different domains to different DNS servers:

```
# Corporate VPN domains
[/corp.company.com/]10.0.0.1

# Local domains
[/local/]192.168.1.1

# Gaming domains to LANCache
[/steamcontent.com/]127.0.0.1:5380
[/epicgames.com/]127.0.0.1:5380

# Everything else to public DNS
#default
```

## Performance Monitoring

### Advanced Netdata Configuration

1. **Install plugins:**
   ```bash
   sudo apt install -y netdata-plugin-chartsd
   ```

2. **Configure alarms:**

   Edit `/etc/netdata/health.d/custom.conf`:
   ```yaml
   alarm: high_cpu_usage
      on: system.cpu
   lookup: average -3m unaligned of user,system,softirq,irq,guest
    units: %
    every: 1m
     warn: $this > 80
     crit: $this > 95
     info: system CPU usage is high
   
   alarm: disk_space_low
      on: disk.space
   lookup: average -1m percentage of used
    units: %
    every: 1m
     warn: $this > 80
     crit: $this > 90
     info: disk space is low
   
   alarm: high_network_usage
      on: system.net
   lookup: average -3m unaligned absolute of received,sent
    units: kilobits/s
    every: 1m
     warn: $this > 80000
     info: network usage is high
   ```

### Custom Monitoring Dashboard

Create `/var/www/html/dashboard.php`:

```php
<!DOCTYPE html>
<html>
<head>
    <title>Server Dashboard</title>
    <meta http-equiv="refresh" content="10">
    <style>
        body { font-family: Arial; margin: 20px; }
        .metric { display: inline-block; margin: 10px; padding: 20px; 
                  border: 1px solid #ddd; border-radius: 5px; }
        .value { font-size: 2em; font-weight: bold; }
        .label { color: #666; }
    </style>
</head>
<body>
    <h1>Home Server Dashboard</h1>
    
    <?php
    // Modem status
    $signal = shell_exec("sudo mmcli -m 0 | grep 'signal quality' | awk '{print $3}'");
    echo "<div class='metric'><div class='value'>$signal</div><div class='label'>Signal Quality</div></div>";
    
    // WiFi clients
    $clients = shell_exec("iw dev wlan0 station dump | grep Station | wc -l");
    echo "<div class='metric'><div class='value'>$clients</div><div class='label'>WiFi Clients</div></div>";
    
    // Cache size
    $cache = shell_exec("du -sh /cache/lancache-data | awk '{print $1}'");
    echo "<div class='metric'><div class='value'>$cache</div><div class='label'>Cache Size</div></div>";
    
    // System load
    $load = sys_getloadavg()[0];
    echo "<div class='metric'><div class='value'>$load</div><div class='label'>System Load</div></div>";
    ?>
</body>
</html>
```

## Security Enhancements

### Intrusion Detection with Suricata

1. **Install Suricata:**
   ```bash
   sudo apt install -y suricata
   ```

2. **Configure:**
   ```bash
   sudo nano /etc/suricata/suricata.yaml
   ```

3. **Update rules:**
   ```bash
   sudo suricata-update
   sudo systemctl enable suricata
   sudo systemctl start suricata
   ```

### VPN Server Setup

#### Using WireGuard

1. **Install WireGuard:**
   ```bash
   sudo apt install -y wireguard
   ```

2. **Generate keys:**
   ```bash
   wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
   ```

3. **Configure server:**
   ```bash
   sudo nano /etc/wireguard/wg0.conf
   ```
   
   ```ini
   [Interface]
   PrivateKey = <server_private_key>
   Address = 10.0.0.1/24
   ListenPort = 51820
   
   PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o wwan0 -j MASQUERADE
   PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o wwan0 -j MASQUERADE
   
   [Peer]
   PublicKey = <client_public_key>
   AllowedIPs = 10.0.0.2/32
   ```

4. **Enable:**
   ```bash
   sudo systemctl enable wg-quick@wg0
   sudo systemctl start wg-quick@wg0
   ```

## Backup and Recovery

### Automated Backup to Cloud

#### Using rclone

1. **Install rclone:**
   ```bash
   curl https://rclone.org/install.sh | sudo bash
   ```

2. **Configure remote:**
   ```bash
   rclone config
   # Follow prompts to setup cloud storage
   ```

3. **Create backup script:**
   
   `/usr/local/bin/backup-to-cloud.sh`:
   ```bash
   #!/bin/bash
   
   DATE=$(date +%Y%m%d)
   BACKUP_DIR="/tmp/backup-$DATE"
   
   mkdir -p $BACKUP_DIR
   
   # Backup configs
   tar -czf $BACKUP_DIR/configs.tar.gz \
       /etc/hostapd/ \
       /etc/netplan/ \
       /opt/AdGuardHome/
   
   # Upload to cloud
   rclone copy $BACKUP_DIR remote:backups/
   
   # Cleanup
   rm -rf $BACKUP_DIR
   ```

4. **Schedule with cron:**
   ```bash
   sudo crontab -e
   # 0 3 * * 0 /usr/local/bin/backup-to-cloud.sh
   ```

## Additional Resources

### Recommended Reading
- Linux networking fundamentals
- Docker orchestration with Compose
- Network security best practices
- WiFi 7 specifications (IEEE 802.11be)
- 5G cellular technology

### Tools for Advanced Users
- **Wireshark**: Packet analysis
- **iperf3**: Bandwidth testing
- **tcpdump**: Traffic capture
- **ansible**: Automation
- **terraform**: Infrastructure as code

### Performance Benchmarking
```bash
# Network throughput
iperf3 -s  # On server
iperf3 -c 192.168.1.1  # On client

# Disk performance
sudo fio --name=test --size=4G --rw=randread --bs=4k --numjobs=4
sudo hdparm -Tt /dev/nvme0n1

# DNS performance
dnsperf -d queries.txt -s 192.168.1.1

# WiFi analysis
wavemon  # Install: sudo apt install wavemon
```

---

**Remember**: Advanced configurations require careful testing. Always backup before making changes!
