# Network Setup with systemd-networkd

**Recommended Configuration Sequence**
When configuring your network, follow this order:
1. Configure **wired** or **wireless DHCP** first (quickly test connectivity).
2. Configure **static IP** if needed (after confirming DHCP works).
3. Configure **Wi-Fi authentication** with `wpa_supplicant` (for wireless interfaces).
4. Restart `systemd-networkd` to apply changes.
5. Optionally, optimize boot time with `systemd-networkd-wait-online.service`.

## 1. DHCP Network

```ini
# /etc/systemd/network/20-dhcp.network
[Match]
Name=*

[Network]
DHCP=yes
LinkLocalAddressing=no
IPv6AcceptRA=no
```

**Field Explanation**:
| Field                    | Purpose                                           | Notes                                                                                            |
| ------------------------ | ------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `[Match]`                | Matches network interfaces                        | `Name=*` matches all interfaces; can be changed to specific interfaces like `enp3s0` or `wlp2s0` |
| `DHCP=yes`               | Enable DHCP to automatically obtain an IP address | Supports both IPv4 and IPv6                                                                      |
| `LinkLocalAddressing=no` | Disable IPv6 link-local addresses                 | Prevents generating fe80:: addresses                                                             |
| `IPv6AcceptRA=no`        | Accept Router Advertisements for IPv6             | Disable if automatic IPv6 configuration is not needed                                            |


**Use Cases**:
- Wired interfaces (office/home DHCP network)
- Wireless interfaces (after connecting via `wpa_supplicant`)

## 2. Static IP Network

```ini
# /etc/systemd/network/30-static.network
[Match]
Name=*

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=8.8.8.8
LinkLocalAddressing=no
IPv6AcceptRA=no
```

**Field Explanation**:
| Field                      | Purpose                                  | Notes                                                                      |
| -------------------------- | ---------------------------------------- | -------------------------------------------------------------------------- |
| `[Match]`                  | Matches network interfaces               | Can use wildcard `*` or a specific interface name                          |
| `Address=192.168.1.100/24` | Sets static IP address and subnet mask   | `/24` equals 255.255.255.0                                                 |
| `Gateway=192.168.1.1`      | Default gateway                          | Used for packets leaving the local network                                 |
| `DNS=8.8.8.8`              | Specifies DNS server                     | Works in systemd-networkd ≥239; otherwise configure via `systemd-resolved` |
| `LinkLocalAddressing=no`   | Disable IPv6 link-local addresses        | Prevents fe80:: addresses                                                  |
| `IPv6AcceptRA=no`          | Do not accept IPv6 router advertisements | Prevents automatic IPv6 configuration                                      |


**Use Cases**:
- Wired interfaces requiring fixed IP (e.g., servers, NAS)
- Wireless interfaces requiring fixed IP (e.g., internal LAN Wi-Fi devices)

## 3. Wi-Fi Authentication (wpa_supplicant)

### 3.1 Generate configuration

```bash
sudo wpa_passphrase "YourSSID" "YourPassword" | sudo tee /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
```

### 3.2 Verify configuration

```ini
ctrl_interface=/run/wpa_supplicant
network={
    ssid="YourSSID"
    #psk="YourPassword"
    psk=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
}
```

### 3.3 Enable wpa_supplicant

```bash
sudo systemctl enable --now wpa_supplicant@wlp2s0.service
```

**Notes**:
- Wi-Fi authentication is managed separately from DHCP or static IP configuration
- Ensure the interface name (e.g., `wlp2s0`) matches the actual system interface

## 4. Restart networkd service

```bash
sudo systemctl restart systemd-networkd
sudo systemctl status systemd-networkd
```

## 5. Boot Time Optimization

Check slow-start services:
```bash
systemd-analyze blame
```

### Option 1: Disable wait-online globally

```bash
sudo systemctl disable systemd-networkd-wait-online.service
```

### Option 2: Wait for a specific interface only

```bash
sudo systemctl edit systemd-networkd-wait-online.service
```
Add:
```ini
[Service]
ExecStart=
ExecStart=/usr/lib/systemd/systemd-networkd-wait-online --interface=wlp2s0 --timeout=30
```

## 6. Notes

1. Interface names must match actual devices:
   - Wired interfaces usually start with `enp*`
   - Wireless interfaces usually start with `wlp*`
   - Check with `ip link`
2. Do not enable both DHCP and static IP on the same interface simultaneously
3. Restart `systemd-networkd` after modifying configuration files
4. Wi-Fi authentication relies on `wpa_supplicant` and is managed separately

## Summary

- Follow the recommended **configuration sequence**: DHCP → Static → Wi-Fi → restart → boot optimization
- Unified setup for wired/wireless interfaces using `systemd-networkd`
- Wi-Fi authentication is separate for clarity and maintainability
- Clean, maintainable, and supports boot time optimization
