# Arch Journal

A personal Arch Linux setup and configuration journal. This document records optimizations, networking configurations, and commonly used software for easier reproducibility and reference.

---

## System

### Boot Time Optimization

Check which services delay boot:

```bash
systemd-analyze blame
```

Two common approaches to optimize `systemd-networkd-wait-online`:

**Option 1: Disable wait-online globally**

```bash
sudo systemctl disable systemd-networkd-wait-online.service
```

**Option 2: Wait only for a specific interface**

Edit the override configuration:

```bash
sudo systemctl edit systemd-networkd-wait-online.service
```

Add the following:

```ini
[Service]
ExecStart=
ExecStart=/usr/lib/systemd/systemd-networkd-wait-online \
  --interface=wlp2s0 --timeout=30
```

---

## Services

### systemd-networkd

#### DHCP Network

```ini
# /etc/systemd/network/20-dhcp.network
[Match]
Name=*

[Network]
DHCP=yes
LinkLocalAddressing=no
IPv6AcceptRA=no
```

#### Static IP Network

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

#### Wi-Fi Authentication

1. Generate configuration:

   ```bash
   sudo wpa_passphrase "YourSSID" "YourPassword" | \
     sudo tee /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
   ```

2. Verify configuration:

   ```ini
   ctrl_interface=/run/wpa_supplicant
   network={
       ssid="YourSSID"
       #psk="YourPassword"
       psk=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
   }
   ```

3. Enable `wpa_supplicant`:

   ```bash
   sudo systemctl enable --now wpa_supplicant@wlp2s0.service
   ```

4. Enable `systemd-networkd`:

   ```bash
   sudo systemctl enable --now systemd-networkd
   ```

### systemd-resolved

Enable and start the service:

```bash
sudo systemctl enable --now systemd-resolved
```

---

## Software

### Docker

Install Docker and configure user permissions:

```bash
sudo pacman -S docker
sudo usermod -aG docker $(whoami)
sudo mkdir /etc/docker
```

Set up a registry mirror:

```bash
cat <<EOF | sudo tee /etc/docker/daemon.json > /dev/null
{
    "registry-mirrors" : [
        "https://docker.1panel.live"
    ]
}
EOF
```

Enable and start Docker:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now docker
```

### Docker Compose

Install Docker Compose:

```bash
sudo pacman -S docker-compose
```

### v2RayA

Prepare directories and compose file:

```bash
mkdir -p Software/v2raya/data
nvim Software/v2raya/docker-compose.yml
```

Example configuration:

```yaml
services:
  v2raya:
    image: mzz2017/v2raya:v2.2.5.8
    restart: always
    privileged: true
    network_mode: host
    container_name: v2raya
    environment:
      V2RAYA_LOG_FILE: /tmp/v2raya.log
      V2RAYA_V2RAY_BIN: /usr/local/bin/xray
      V2RAYA_NFTABLES_SUPPORT: off
      IPTABLES_MODE: legacy
    volumes:
      - /lib/modules:/lib/modules:ro
      - /etc/resolv.conf:/etc/resolv.conf
      - $PWD/data:/etc/v2raya
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
```

Start the container:

```bash
cd Software/v2raya
docker-compose up -d
```
