# Dell Precision T7810 Home Server Setup Guide

Transform your Dell Precision T7810 workstation into a powerful home server with 5G cellular internet, WiFi 7 access point, ad blocking, and game caching capabilities.

## Overview

This repository contains comprehensive guides for converting a Dell Precision T7810 (2x Xeon E5-2620 v3, 32GB RAM, 500GB HDD) into a feature-rich home server with:

- **5G Cellular Modem (RM520)**: High-speed internet via cellular connection
- **WiFi 7 Access Point**: Latest generation wireless networking for your home
- **AdGuard Home**: Network-wide ad blocking and DNS filtering
- **LANCache**: Game and software update caching to save bandwidth
- **Full routing and NAT**: Complete home network solution

## Features

✅ **High-Speed Connectivity**: 5G cellular modem with failover support  
✅ **WiFi 7 Technology**: Multi-gigabit wireless speeds with latest 802.11be standard  
✅ **Ad Blocking**: Network-wide ad and tracker blocking with AdGuard Home  
✅ **Bandwidth Saving**: LANCache for Steam, Epic Games, Origin, and more  
✅ **Professional Monitoring**: System health and network monitoring tools  
✅ **Security Hardened**: Firewall, fail2ban, and security best practices  
✅ **Fully Open Source**: Uses Ubuntu Server 24.04 LTS and open-source software  

## Documentation

### Quick Start
- **[QUICK_START.md](QUICK_START.md)** - Get up and running in a few hours with step-by-step instructions

### Detailed Guides
- **[HARDWARE_GUIDE.md](HARDWARE_GUIDE.md)** - Complete hardware compatibility list, PCIe slot allocation, and shopping list
- **[SOFTWARE_SETUP.md](SOFTWARE_SETUP.md)** - Detailed software installation and configuration for all services
- **[NETWORK_ARCHITECTURE.md](NETWORK_ARCHITECTURE.md)** - Network topology, IP addressing, DNS flow, and architecture diagrams

## Hardware Requirements

### Base System (What You Have)
- Dell Precision T7810 Workstation
- 2x Intel Xeon E5-2620 v3 (12 cores / 24 threads total)
- 32GB DDR4 ECC RAM
- 500GB HDD
- Multiple PCIe slots (3x x16, 2x x4)

### Required Hardware Additions (~$600-1,100)

#### Essential Components:
1. **5G Modem**: Quectel RM520N-GL module + M.2 to PCIe adapter + antennas (~$200-300)
2. **WiFi 7 Card**: Intel BE200 or Fenvi FE1BE (~$50-150)
3. **NVMe SSD**: 1TB for OS and system (~$80-120)
4. **Cache Storage**: 4TB SSD/HDD for LANCache (~$150-400)

#### Optional Enhancements:
- Intel I350-T4 quad-port gigabit NIC (~$100-150)
- Additional antennas and cables (~$30-50)
- Cooling enhancements (~$20-50)

See **[HARDWARE_GUIDE.md](HARDWARE_GUIDE.md)** for detailed specifications and shopping list.

## Software Stack

- **Operating System**: Ubuntu Server 24.04 LTS
- **Modem Management**: ModemManager + NetworkManager
- **WiFi Access Point**: hostapd with WiFi 7 support
- **DNS/Ad Blocking**: AdGuard Home
- **DHCP Server**: dnsmasq
- **Game Caching**: LANCache (Docker)
- **Monitoring**: Netdata
- **Containerization**: Docker + Docker Compose

## Network Architecture

```
┌─────────────────┐
│  5G Cellular    │
│   Internet      │
│   (RM520)       │
└────────┬────────┘
         │ wwan0
┌────────▼────────┐
│  T7810 Server   │
│  192.168.1.1    │
│                 │
│  ┌──────────┐   │
│  │AdGuard   │   │
│  │LANCache  │   │
│  │Routing   │   │
│  └──────────┘   │
└────────┬────────┘
         │ wlan0 (WiFi 7)
┌────────▼────────┐
│  Home Network   │
│  192.168.1.0/24 │
│                 │
│  Clients, IoT,  │
│  Gaming Devices │
└─────────────────┘
```

See **[NETWORK_ARCHITECTURE.md](NETWORK_ARCHITECTURE.md)** for detailed diagrams and flow charts.

## Key Benefits

### Cost Savings
- **No ISP fees**: Use cellular data plan instead of traditional broadband
- **No separate AP**: Integrated WiFi 7 access point
- **Bandwidth savings**: LANCache reduces cellular data usage by 50-70%
- **No monthly router rental**: Own your infrastructure

### Performance
- **5G speeds**: Up to 1-4 Gbps download (depending on coverage)
- **WiFi 7**: Multi-gigabit wireless speeds
- **Low latency**: Optimized for gaming and real-time applications
- **LANCache**: Near-instant game updates from local cache

### Features
- **Ad-free network**: Block ads, trackers, and malware at DNS level
- **Parental controls**: Content filtering via AdGuard
- **Monitoring**: Real-time network and system monitoring
- **Customizable**: Full control over your network

## Installation Time

- **Hardware installation**: 1-2 hours
- **Software setup**: 2-3 hours
- **Testing and optimization**: 1-2 hours
- **Total**: ~4-7 hours for complete setup

Follow the **[QUICK_START.md](QUICK_START.md)** guide for fastest setup.

## Use Cases

### Perfect For:
- **Cellular-only internet**: Areas without cable/fiber availability
- **Gaming households**: Multiple PCs/consoles downloading large games
- **Privacy-conscious users**: Network-wide ad and tracker blocking
- **Tech enthusiasts**: Learn networking, Linux, and self-hosting
- **Home labs**: Experiment with enterprise technologies

### Ideal Performance Scenarios:
- Homes with good 5G coverage
- 2-10 connected devices simultaneously
- Gaming (Steam, Epic, Origin, Battle.net, etc.)
- 4K video streaming (2-3 streams)
- General web browsing and productivity

## Technical Highlights

### Cellular Modem (RM520)
- 5G NR Sub-6GHz support
- Dual SIM with automatic failover
- Up to 4 Gbps download speeds
- Compatible with major US carriers

### WiFi 7 (802.11be)
- 2.4 GHz, 5 GHz, and 6 GHz bands
- Up to 5.8 Gbps theoretical speeds
- MU-MIMO, OFDMA, and beamforming
- WPA3 security with SAE

### AdGuard Home
- DNS-level ad blocking
- 300K+ blocked domains
- Custom filtering rules
- Encrypted DNS (DNS-over-HTTPS)
- Query logging and statistics

### LANCache
- Supports 40+ CDNs
- Steam, Epic, Origin, Blizzard, Ubisoft
- Windows Updates caching
- Automatic cache management
- Transparent to clients

## Prerequisites Knowledge

### Recommended Skills:
- Basic Linux command line
- Understanding of networking concepts (IP, DNS, DHCP)
- Comfortable with terminal/SSH
- Ability to follow technical documentation

### You'll Learn:
- Advanced networking (routing, NAT, VLANs)
- Linux system administration
- Docker containerization
- DNS and DHCP configuration
- Network security best practices

## Support and Troubleshooting

Each guide includes comprehensive troubleshooting sections:
- Modem connectivity issues
- WiFi configuration problems
- DNS resolution failures
- LANCache hit rate optimization
- Performance tuning tips

## Community and Resources

### Official Documentation
- [AdGuard Home Wiki](https://github.com/AdguardTeam/AdGuardHome/wiki)
- [LANCache Documentation](https://lancache.net/)
- [Ubuntu Server Guide](https://ubuntu.com/server/docs)
- [hostapd Documentation](https://w1.fi/hostapd/)

### Community Support
- r/homelab - Home server enthusiasts
- r/selfhosted - Self-hosting discussions
- r/homeservers - Home server projects
- LANCache Discord - Active support community

## Contributing

Have improvements or suggestions? Contributions are welcome:
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

This documentation is provided as-is for educational and personal use.

## Disclaimer

- Cellular data plans have limits - monitor your usage
- WiFi 7 requires compatible client devices for full speeds
- Performance varies based on cellular coverage
- Some assembly required - this is a DIY project
- Follow local regulations for wireless frequencies

## Getting Started

1. **Review**: Read through [HARDWARE_GUIDE.md](HARDWARE_GUIDE.md) to understand requirements
2. **Order**: Purchase necessary hardware components
3. **Install**: Follow [QUICK_START.md](QUICK_START.md) for installation
4. **Configure**: Use [SOFTWARE_SETUP.md](SOFTWARE_SETUP.md) for detailed configuration
5. **Optimize**: Reference [NETWORK_ARCHITECTURE.md](NETWORK_ARCHITECTURE.md) for advanced tuning

## FAQ

**Q: Can I use this with regular broadband instead of cellular?**  
A: Yes! Simply skip the RM520 modem setup and use ethernet for WAN.

**Q: Will this work with WiFi 6/6E devices?**  
A: Yes, WiFi 7 is backwards compatible with WiFi 6/6E/5/4.

**Q: How much cellular data will I use?**  
A: Varies by usage. LANCache significantly reduces data for game downloads. Budget 100-500GB/month for typical household.

**Q: Can I use Windows instead of Linux?**  
A: Possible but not recommended. Linux provides better performance and open-source tools.

**Q: What if I don't have good 5G coverage?**  
A: The RM520 also supports 4G LTE. Performance will be lower but still functional.

**Q: Is this better than a commercial router?**  
A: More flexible and powerful, but requires more setup. Great for learning and customization.

## System Requirements Summary

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 2x Xeon E5-2620 v3 | Same (sufficient) |
| RAM | 32GB | 32GB+ |
| OS Storage | 256GB SSD | 1TB NVMe SSD |
| Cache Storage | 1TB HDD | 4TB SSD |
| Network | WiFi 7 card | WiFi 7 + extra NIC |
| Internet | 4G LTE | 5G NR |

## Project Status

✅ Hardware selection and compatibility verified  
✅ Software stack tested and documented  
✅ Network architecture designed  
✅ Security hardening implemented  
✅ Monitoring and management tools integrated  
✅ Comprehensive documentation complete  

## Acknowledgments

Built using excellent open-source projects:
- Ubuntu Server
- AdGuard Team
- LANCache.net community
- hostapd developers
- Quectel for modem specifications

---

**Ready to build your home server?** Start with **[QUICK_START.md](QUICK_START.md)**!