---
title: Server Environment & Network
date: 2026-02-24
tags: [server, hardware, network, proxy, mihomo, tailscale]
depends_on: []
status: current
last_updated: 2026-02-24
---

# Server Environment - zhuangbaopenclaw

## Hardware
| Component | Spec | Limitation |
|-----------|------|-----------|
| **CPU** | Intel i5-7300HQ @ 2.50GHz, 4C/4T | No hyperthreading, VT-x disabled in BIOS |
| **RAM** | 7.6 GB (DDR4) | ~6 GB available after services |
| **GPU** | GTX 1050 Mobile 2GB VRAM | Uses nouveau driver, NO CUDA |
| **Disk** | 116 GB NVMe SSD | ~105 GB free |
| **OS** | Ubuntu 24.04 LTS, kernel 6.8 | |

## Network
| Interface | Address | Notes |
|-----------|---------|-------|
| **LAN (WiFi)** | 192.168.0.18 | Primary access from WSL |
| **Tailscale** | 100.79.146.9 | For remote access outside LAN |
| **Hostname** | zhuangbaopenclaw | |

## Proxy (mihomo)
- **Purpose**: Server in mainland China, needs proxy for Telegram API, GitHub, and AI API endpoints
- **Software**: mihomo v1.19.8 (Clash Meta)
- **Config**: `/etc/mihomo/config.yaml`
- **Subscription**: Speed Cloud (速云), Trojan protocol, expires 2026-05-17
- **Local port**: `127.0.0.1:7890` (mixed HTTP/SOCKS)
- **Systemd**: `mihomo.service` (system-level, enabled, auto-start)
- **GeoIP data**: Manually downloaded to `/etc/mihomo/` (country.mmdb, geoip.dat, geosite.dat)

### Subscription Update Command
```bash
sudo curl -sL -A 'ClashForWindows/0.20.39' \
  'https://kudun.xn--vuqy1f6uqyhu2rlsp5a.com/login/sy/api/v1/client/subscribe?token=TOKEN' \
  -o /etc/mihomo/config.yaml && sudo systemctl restart mihomo
```

### OpenClaw Proxy Integration
Gateway reads proxy from systemd environment override:
```
# ~/.config/systemd/user/openclaw-gateway.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,192.168.0.0/16"
```

## Services Running
| Service | Type | Port | Auto-start |
|---------|------|------|-----------|
| mihomo | system | 7890 | yes |
| openclaw-gateway | user | 18789 | yes (lingering) |
| TigerVNC :1 | user | 5901 | yes |
| noVNC/websockify | user | 6080 | yes (lingering) |
| XFCE4 desktop | user | - | via VNC |

## Local Model Capability
**Not recommended on this hardware:**
- GPU has no CUDA (nouveau driver only)
- 2GB VRAM limits to ≤1.5B parameter models at Q4
- CPU-only inference would be very slow
- Best to continue using cloud API via proxy

## SSH Access from WSL
```bash
ssh -i ~/.ssh/id_ed25519 zhuangba@192.168.0.18
```
