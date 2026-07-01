# Global DNS Configuration Guide

Date: 2026-07-01
System: CachyOS / Arch-based Linux, systemd-resolved + NetworkManager

## Overview

This guide explains how to configure system-wide DNS with Cloudflare as primary and Google as secondary DNS servers, with DNS over TLS enabled for encrypted queries. The configuration applies globally to all current and future network connections.

## Why Use Custom DNS?

| Aspect | Router/ISP DNS | Cloudflare/Google |
|--------|----------------|-------------------|
| **Speed** | Depends on ISP (often slower) | Optimized globally (faster) |
| **Privacy** | ISP can log your queries | Cloudflare: no logging, audited yearly |
| **Reliability** | Can go down with ISP issues | Redundant infrastructure |
| **Security** | Basic | Optional malware blocking |

## DNS Providers

| Role | Provider | IPv4 | IPv6 |
|------|----------|------|------|
| Primary | Cloudflare | `1.1.1.1` | `2606:4700:4700::1111` |
| Secondary | Google | `8.8.8.8` | `2001:4860:4860::8888` |
| Fallback | Quad9 | `9.9.9.9` | `2620:fe::9` |

## Prerequisites

- Arch-based Linux distribution (CachyOS)
- systemd-resolved installed and running
- NetworkManager installed and running
- Root/sudo access

## Configuration Steps

### 1. Configure systemd-resolved

Edit `/etc/systemd/resolved.conf` to set global DNS servers and enable DNS over TLS:

```ini
[Resolve]
# Primary: Cloudflare, Fallback: Google
DNS=1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google 2606:4700:4700::1111#cloudflare-dns.com 2001:4860:4860::8888#dns.google
# Fallback: Quad9 (security-focused)
FallbackDNS=9.9.9.9#dns.quad9.net 2620:fe::9#dns.quad9.net
#Domains=
#DNSSEC=no
DNSOverTLS=yes
#MulticastDNS=yes
#LLMNR=yes
#Cache=yes
#CacheFromLocalhost=no
#DNSCacheSize=4096
```

Key settings:
- `DNS=` - Primary DNS servers (Cloudflare and Google)
- `FallbackDNS=` - Used when primary servers are unreachable
- `DNSOverTLS=yes` - Encrypts DNS queries (prevents ISP snooping)

### 2. Configure NetworkManager Global Defaults

Create `/etc/NetworkManager/conf.d/99-dns-defaults.conf` to apply DNS settings to all connections:

```ini
# Global DNS defaults for all NetworkManager connections
# Cloudflare primary, Google secondary
[global-dns]
search=lan

[global-dns-domain-*]
servers=1.1.1.1,8.8.8.8,2606:4700:4700::1111,2001:4860:4860::8888
```

This ensures any new WiFi connection automatically uses these DNS servers.

### 3. Update Existing Connections

Update your current connection to ignore DHCP-assigned DNS:

```bash
sudo nmcli con mod "YourConnectionName" ipv4.dns "1.1.1.1 8.8.8.8"
sudo nmcli con mod "YourConnectionName" ipv4.ignore-auto-dns yes
sudo nmcli con mod "YourConnectionName" ipv6.dns "2606:4700:4700::1111 2001:4860:4860::8888"
sudo nmcli con mod "YourConnectionName" ipv6.ignore-auto-dns yes
```

Replace `"YourConnectionName"` with your actual connection name (check with `nmcli con show`).

### 4. Restart Services

Apply all changes by restarting the services:

```bash
sudo systemctl restart systemd-resolved
sudo systemctl restart NetworkManager
```

## Verification Commands

After configuration, verify the setup:

```bash
# Check systemd-resolved status
resolvectl status

# Expected output should show:
# - Global DNS Servers: 1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google
# - Protocols: +DNSOverTLS
# - Fallback DNS Servers: 9.9.9.9#dns.quad9.net

# Check resolv.conf
cat /etc/resolv.conf
# Should show: nameserver 127.0.0.53

# Test DNS resolution
resolvectl query example.com

# Or using dig
dig example.com +short

# Test with specific server
dig @1.1.1.1 example.com
```

## Troubleshooting

### DNS Resolution Fails After Configuration

1. **Check systemd-resolved is running:**
   ```bash
   systemctl status systemd-resolved
   ```

2. **Verify DNS servers are configured:**
   ```bash
   resolvectl status | grep "DNS Servers"
   ```

3. **Check for errors in logs:**
   ```bash
   journalctl -u systemd-resolved -e
   ```

### NetworkManager Not Applying DNS to New Connections

1. **Verify global config exists:**
   ```bash
   cat /etc/NetworkManager/conf.d/99-dns-defaults.conf
   ```

2. **Check NetworkManager config loading:**
   ```bash
   NetworkManager --print-config
   ```

3. **Restart NetworkManager:**
   ```bash
   sudo systemctl restart NetworkManager
   ```

### DNS Over TLS Not Working

Some networks block port 853 (DNS over TLS). If DNS queries fail:

1. **Check TLS status:**
   ```bash
   resolvectl status | grep DNSOverTLS
   ```

2. **Test TLS connection:**
   ```bash
   openssl s_client -connect 1.1.1.1:853 -servername cloudflare-dns.com
   ```

3. **Temporary workaround** - disable TLS if needed:
   ```bash
   # Edit /etc/systemd/resolved.conf
   DNSOverTLS=opportunistic
   sudo systemctl restart systemd-resolved
   ```

## How It Works

1. All DNS queries route through `127.0.0.53` (systemd-resolved stub resolver)
2. systemd-resolved forwards queries to Cloudflare/Google over encrypted TLS
3. NetworkManager applies these DNS settings to all connections globally
4. Any new WiFi connection automatically uses the configured DNS servers

## Key Files Modified

- `/etc/systemd/resolved.conf` - Global DNS servers and DNS over TLS
- `/etc/NetworkManager/conf.d/99-dns-defaults.conf` - NetworkManager global DNS defaults (new file)
- NetworkManager connection profiles - Updated to ignore DHCP DNS

## Summary

The DNS configuration provides:
1. Cloudflare as primary DNS (fast, privacy-focused, no logging)
2. Google as secondary DNS (reliable, good global infrastructure)
3. Quad9 as fallback (security-focused, blocks malicious domains)
4. DNS over TLS enabled (encrypted queries prevent ISP snooping)
5. Global application to all current and future network connections

## References

- [systemd-resolved Documentation](https://man.archlinux.org/man/systemd-resolved.8.en)
- [resolved.conf Configuration](https://man.archlinux.org/man/resolved.conf.5.en)
- [Arch Wiki - systemd-resolved](https://wiki.archlinux.org/title/systemd-resolved)
- [Cloudflare DNS](https://1.1.1.1/)
- [Google Public DNS](https://developers.google.com/speed/public-dns)
- [Quad9 DNS](https://quad9.net/)

## Changelog

- **2026-07-01**: Initial documentation
  - Configured Cloudflare as primary DNS
  - Configured Google as secondary DNS
  - Added Quad9 as fallback
  - Enabled DNS over TLS
  - Applied global DNS settings to all NetworkManager connections
