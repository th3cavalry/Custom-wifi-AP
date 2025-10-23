# Network Architecture and Design

## Network Topology Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    INTERNET (5G Cellular)                    │
│                         RM520 Modem                          │
└────────────────────────────┬────────────────────────────────┘
                             │ wwan0
                             │
┌────────────────────────────▼────────────────────────────────┐
│                  Dell Precision T7810                        │
│                    Home Server/Router                        │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Routing   │  │  AdGuard     │  │  LANCache    │       │
│  │   & NAT     │  │  Home        │  │  (Docker)    │       │
│  │  (iptables) │  │  (DNS/DHCP)  │  │              │       │
│  └─────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  IP: 192.168.1.1                                            │
└────────────────────────────┬────────────────────────────────┘
                             │ wlan0 (WiFi 7)
                             │
┌────────────────────────────▼────────────────────────────────┐
│                    Home WiFi Network                         │
│                  192.168.1.0/24 LAN                         │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Phones  │  │ Laptops  │  │  Gaming  │  │   IoT    │   │
│  │          │  │          │  │  Console │  │  Devices │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                              │
│  DHCP Range: 192.168.1.50 - 192.168.1.250                  │
└─────────────────────────────────────────────────────────────┘
```

## IP Address Scheme

### Server/Gateway
- **Gateway IP**: 192.168.1.1
- **Subnet**: 255.255.255.0 (/24)
- **Network**: 192.168.1.0/24
- **Usable IPs**: 192.168.1.1 - 192.168.1.254

### Address Allocation

| Range | Purpose | Details |
|-------|---------|---------|
| 192.168.1.1 | Server/Gateway | Main server IP |
| 192.168.1.2-49 | Static/Reserved | For servers, printers, NAS, etc. |
| 192.168.1.50-250 | DHCP Pool | Automatic assignment for clients |
| 192.168.1.251-254 | Reserved | Future use |

### Recommended Static Assignments

| IP | Device | Purpose |
|----|--------|---------|
| 192.168.1.1 | T7810 Server | Gateway, DNS, WiFi AP |
| 192.168.1.10 | Management Interface | Server management (if separate NIC) |
| 192.168.1.20 | Network Printer | (If applicable) |
| 192.168.1.21 | NAS Device | (If applicable) |
| 192.168.1.30 | Gaming Console 1 | Static for port forwarding |
| 192.168.1.31 | Gaming Console 2 | Static for port forwarding |
| 192.168.1.40 | Smart TV | Static for reliable streaming |

## Service Port Mapping

### Server Services

| Service | Port(s) | Protocol | Access | Purpose |
|---------|---------|----------|--------|---------|
| SSH | 22 | TCP | LAN Only | Server management |
| DNS | 53 | TCP/UDP | LAN | AdGuard Home DNS |
| DHCP | 67 | UDP | LAN | IP address assignment |
| HTTP | 80 | TCP | LAN | LANCache service |
| HTTPS | 443 | TCP | LAN | LANCache service |
| AdGuard UI | 3000 | TCP | LAN Only | Web interface |
| Netdata | 19999 | TCP | LAN Only | Monitoring dashboard |
| LANCache DNS | 5380 | TCP/UDP | Localhost | Internal LANCache DNS |

### Port Forwarding (Optional for Gaming)

| Service | External Port | Internal IP | Internal Port | Protocol |
|---------|---------------|-------------|---------------|----------|
| Gaming Server | 27015 | 192.168.1.30 | 27015 | TCP/UDP |
| Minecraft | 25565 | 192.168.1.31 | 25565 | TCP |
| Example | 3074 | 192.168.1.30 | 3074 | TCP/UDP |

Configure in iptables:
```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 27015 -j DNAT --to-destination 192.168.1.30:27015
sudo iptables -t nat -A PREROUTING -p udp --dport 27015 -j DNAT --to-destination 192.168.1.30:27015
```

## DNS Flow Diagram

```
┌────────────┐
│   Client   │
│  Device    │
└─────┬──────┘
      │ DNS Query (example.com)
      ▼
┌─────────────────┐
│  AdGuard Home   │ ─────► Block if in blocklist
│  192.168.1.1:53 │        (ads, malware, etc.)
└─────┬───────────┘
      │ Not blocked
      │
      ├─► Is it a gaming domain? ───Yes──► ┌─────────────────┐
      │   (e.g., steamcontent.com)         │  LANCache DNS   │
      │                                     │  127.0.0.1:5380 │
      │                                     └────────┬────────┘
      │                                              │
      │                                              ▼
      │                                     ┌─────────────────┐
      │                                     │ Return server   │
      │                                     │ IP: 192.168.1.1 │
      │                                     └─────────────────┘
      │
      └─► Regular domain? ────────────► ┌──────────────────┐
                                        │  Upstream DNS    │
                                        │  (Cloudflare,    │
                                        │   Google, etc.)  │
                                        └────────┬─────────┘
                                                 │
                                                 ▼
                                        ┌──────────────────┐
                                        │ Return actual IP │
                                        └──────────────────┘
```

## LANCache Flow Diagram

```
Client requests game download (e.g., Steam game)

┌────────────┐
│   Client   │
│  Device    │
└─────┬──────┘
      │ DNS: steamcontent.com
      ▼
┌─────────────────┐
│  AdGuard Home   │ ──────► LANCache DNS
└─────┬───────────┘         (domain matched)
      │
      │ Returns: 192.168.1.1
      ▼
┌────────────┐
│   Client   │ ──────► HTTP GET steamcontent.com/game.bin
└─────┬──────┘         (to 192.168.1.1)
      │
      ▼
┌────────────────────────┐
│  LANCache (Nginx)      │
│  Check if cached?      │
└────────┬───────────────┘
         │
         ├─► YES (Cache Hit) ────► Serve from local cache
         │                          Ultra-fast delivery
         │                          No internet bandwidth used
         │
         └─► NO (Cache Miss) ──────► Download from internet
                                      Save to cache
                                      Serve to client
                                      Future requests = instant
```

## WiFi Configuration

### WiFi 7 Settings

| Parameter | Value | Notes |
|-----------|-------|-------|
| SSID | YourHomeNetwork | Change to preferred name |
| Password | Strong passphrase | WPA3-SAE, min 12 chars |
| Frequency | 5GHz (primary) | Better range/speed balance |
| Channel | 36-48 (5GHz) | Auto or manual selection |
| Channel Width | 160MHz | Maximum WiFi 7 performance |
| Security | WPA3-SAE | Most secure, backwards compatible |
| Max Clients | ~50-100 | Depends on card and load |

### 2.4GHz vs 5GHz vs 6GHz

| Band | Pros | Cons | Best For |
|------|------|------|----------|
| 2.4GHz | Better range, better penetration | Slower, more interference | IoT, smart home devices |
| 5GHz | Good speed, less interference | Less range than 2.4GHz | General use, streaming |
| 6GHz | Highest speed, no legacy interference | Shortest range, limited device support | Gaming, high bandwidth |

### Recommended Configuration

For maximum compatibility and performance:

1. **Primary SSID**: 5GHz band (channel 36-48)
   - Use for most devices
   - Best balance of speed and range

2. **Secondary SSID** (optional): 2.4GHz band
   - For IoT devices and older equipment
   - Use separate SSID to avoid band steering issues

3. **6GHz**: Enable if you have WiFi 7 devices
   - Automatic fallback to 5GHz for unsupported devices

## QoS (Quality of Service) Configuration

### Priority Classes

| Priority | Usage | Examples |
|----------|-------|----------|
| High | Interactive gaming, VoIP | Online gaming, video calls |
| Medium | Streaming, browsing | Netflix, YouTube, web |
| Low | Downloads, updates | Game downloads (via LANCache), backups |

### Implementation

Using `tc` (traffic control):

```bash
# Example QoS configuration
# Create root qdisc
tc qdisc add dev wlan0 root handle 1: htb default 30

# Create classes
tc class add dev wlan0 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit
tc class add dev wlan0 parent 1:1 classid 1:10 htb rate 500mbit ceil 1000mbit prio 1  # High
tc class add dev wlan0 parent 1:1 classid 1:20 htb rate 300mbit ceil 800mbit prio 2   # Medium
tc class add dev wlan0 parent 1:1 classid 1:30 htb rate 200mbit ceil 600mbit prio 3   # Low

# Add filters (example for gaming ports)
tc filter add dev wlan0 protocol ip parent 1:0 prio 1 u32 match ip dport 27015 0xffff flowid 1:10
```

## Security Architecture

### Firewall Rules Overview

```
Internet (Cellular)
    │
    ▼
┌────────────────┐
│  INPUT rules   │ ──► Drop all except:
│  (to server)   │     - SSH (from LAN)
└────────────────┘     - DNS/DHCP (from LAN)
                        - Established connections
    │
    ▼
┌────────────────┐
│ FORWARD rules  │ ──► Allow:
│ (through NAT)  │     - LAN → WAN (with NAT)
└────────────────┘     - Established WAN → LAN
                        - Port forwards (if configured)
    │
    ▼
┌────────────────┐
│ OUTPUT rules   │ ──► Allow all
│ (from server)  │     (server-initiated traffic)
└────────────────┘
```

### Security Zones

| Zone | Networks | Trust Level | Rules |
|------|----------|-------------|-------|
| WAN | wwan0 (Cellular) | Untrusted | Drop all incoming |
| LAN | wlan0, eth0 | Trusted | Allow most traffic |
| Server | localhost | Full trust | All access |

### AdGuard Protection Layers

1. **DNS Filtering**: Block malicious domains
2. **Ad Blocking**: Remove advertisements
3. **Tracker Blocking**: Prevent tracking
4. **Parental Controls**: (Optional) Content filtering
5. **Safe Search**: (Optional) Force safe search
6. **Safe Browsing**: Block malware/phishing

## Monitoring and Logging

### Log Locations

| Service | Log Path | Purpose |
|---------|----------|---------|
| System | `/var/log/syslog` | General system logs |
| AdGuard | `/opt/AdGuardHome/data/` | DNS queries, blocks |
| LANCache | `/cache/lancache-logs/` | Cache hits/misses |
| hostapd | `journalctl -u hostapd` | WiFi AP events |
| ModemManager | `journalctl -u ModemManager` | Cellular connection |
| Docker | `docker logs lancache` | Container logs |

### Metrics to Monitor

#### System Health
- CPU usage and temperature
- RAM usage
- Disk I/O and space
- Network throughput

#### Network Performance
- Cellular signal strength
- Data usage (upload/download)
- WiFi client count
- Network latency

#### Service Performance
- DNS query response time
- AdGuard block rate
- LANCache hit rate
- Cache disk usage

## Bandwidth Management

### Expected Usage Patterns

| Activity | Bandwidth | LANCache Benefit |
|----------|-----------|------------------|
| Web browsing | 1-10 Mbps | None |
| Video streaming (1080p) | 5-8 Mbps | None |
| Video streaming (4K) | 25-50 Mbps | None |
| Online gaming | 1-3 Mbps | None |
| Game download (first) | Full speed | None (downloads to cache) |
| Game download (cached) | LAN speed | Huge (no internet used) |
| System updates | Varies | High (Windows, Steam, etc.) |

### Data Savings with LANCache

**Example scenario**: 3 PCs downloading 100GB game
- **Without LANCache**: 300GB cellular data used
- **With LANCache**: 100GB cellular data used (66% savings)
  - First download: 100GB from internet → cache
  - Second/third downloads: From cache (LAN only)

## Scalability and Future Expansion

### Current Capacity
- **WiFi clients**: 50-100 simultaneous
- **DHCP addresses**: 200 available
- **Cache storage**: 4TB
- **Network speed**: Cellular dependent

### Expansion Options

#### Short-term (<1 year)
- Add external WiFi 6E/7 access points for coverage
- Increase cache storage to 8TB+
- Add UPS for power protection
- Implement VLANs for network segmentation

#### Medium-term (1-2 years)
- Upgrade to fiber/cable if available
- Add dedicated firewall appliance
- Implement VPN server
- Add network monitoring (Prometheus/Grafana)

#### Long-term (2+ years)
- Migrate to dedicated router + AP setup
- Implement high-availability (failover)
- Add enterprise-grade switches
- Separate services into VMs/containers

### VLAN Design (Future)

| VLAN ID | Name | Subnet | Purpose |
|---------|------|--------|---------|
| 1 | Management | 192.168.1.0/24 | Server management |
| 10 | Main LAN | 192.168.10.0/24 | Trusted devices |
| 20 | Guest | 192.168.20.0/24 | Guest WiFi |
| 30 | IoT | 192.168.30.0/24 | Smart home devices |
| 40 | Gaming | 192.168.40.0/24 | Gaming devices (QoS) |

## Performance Benchmarks

### Expected Performance Metrics

#### Network Speed
- **Cellular (5G)**: 100-1000 Mbps down, 50-200 Mbps up (varies by location/signal)
- **WiFi 7 (5GHz)**: 1000-3000 Mbps (depends on device capability)
- **LAN (Gigabit)**: 940 Mbps typical
- **LANCache**: Near gigabit (limited by disk/network)

#### Latency
- **Cellular**: 20-50ms typical
- **WiFi 7**: 1-5ms
- **DNS (AdGuard)**: <10ms
- **LANCache (hit)**: <5ms

#### DNS Performance
- **AdGuard (cached)**: <1ms
- **AdGuard (upstream)**: 10-50ms
- **Block rate**: 20-40% of queries (typical)

## Troubleshooting Flow Charts

### No Internet Connection

```
Check physical connections
    │
    ├─► Cellular modem LEDs on? ──No──► Check power/connections
    │                             Yes
    │                              │
    ▼                              ▼
Check modem status              Check antenna connections
mmcli -m 0                      Verify SIM card inserted
    │
    ├─► Signal strength? ──Low──► Relocate/improve antennas
    │                      Good
    │                       │
    ▼                       ▼
Check IP assignment       Check routing
ip addr show wwan0        ip route show
    │                       │
    └──────────┬────────────┘
               │
               ▼
        Test connectivity
        ping 8.8.8.8
```

### WiFi Not Working

```
Check hostapd service
systemctl status hostapd
    │
    ├─► Running? ──No──► systemctl start hostapd
    │              Yes
    │               │
    ▼               ▼
Check logs       Verify config
journalctl -u    /etc/hostapd/hostapd.conf
hostapd          Correct interface?
    │            Channel allowed?
    │               │
    ▼               ▼
Check interface  Test with simple config
iw dev          (no WiFi 7, no encryption)
wlan0 exists?        │
    │                ▼
    └────► If works, add features incrementally
```

### Slow Performance

```
Check cellular signal
mmcli -m 0 | grep signal
    │
    ├─► <50%? ──Yes──► Improve signal/antennas
    │           No
    │            │
    ▼            ▼
Check WiFi      Monitor bandwidth
client signal   iftop -i wwan0
    │               │
    ├─► Weak? ───►  Move device closer
    │    OK         Congestion?
    │    │              │
    │    ▼              ▼
    │  Check for    QoS needed?
    │  interference  Limit devices?
    │    │              │
    └────┴──────────────┘
           │
           ▼
    Check server load
    htop, iotop
    Disk I/O issue?
```

## Best Practices

### Daily Operations
1. Monitor cellular data usage
2. Check system status dashboard
3. Verify all services running
4. Review AdGuard query logs for issues

### Network Management
1. Use static IPs for critical devices
2. Label devices in AdGuard for easy identification
3. Regularly review connected WiFi clients
4. Monitor bandwidth-heavy devices

### Cache Management
1. Monitor LANCache hit rate
2. Adjust cache size based on usage
3. Periodically review cached content
4. Clear old/unused cache entries

### Security
1. Keep system updated
2. Review AdGuard block lists monthly
3. Monitor for unusual DNS queries
4. Check firewall logs regularly
5. Use strong, unique passwords
6. Enable 2FA where possible

### Backup Strategy
1. Daily: Configuration backups
2. Weekly: AdGuard settings/filters
3. Monthly: Full system backup
4. Test restore procedures quarterly

## Resources and References

### Official Documentation
- **Ubuntu Server**: https://ubuntu.com/server/docs
- **AdGuard Home**: https://github.com/AdguardTeam/AdGuardHome/wiki
- **LANCache**: https://lancache.net/docs/
- **hostapd**: https://w1.fi/hostapd/

### Community Resources
- **r/homelab**: Reddit community for home server enthusiasts
- **r/selfhosted**: Self-hosting services discussions
- **LANCache Discord**: Active community support
- **Level1Techs Forum**: Advanced networking discussions

### Useful Tools
- **Fing**: Network scanner mobile app
- **WiFi Analyzer**: Channel selection tool
- **Speedtest CLI**: Bandwidth testing
- **iperf3**: Network performance testing
- **tcpdump**: Packet capture and analysis

## Change Log

When making changes to your network, document them:

| Date | Change | Reason | Impact |
|------|--------|--------|--------|
| YYYY-MM-DD | Initial setup | New deployment | - |
| | | | |
| | | | |

## Contact and Support

Document your configuration specifics:
- **ISP/Carrier**: _____________
- **Data Plan**: _____________
- **WiFi Password**: (Store securely)
- **AdGuard Admin**: _____________
- **Emergency Contact**: _____________
