# Home Lab: Raspberry Pi 5 Home Security Server (Pi-hole + WireGuard VPN)

**Date:** October 25, 2026

## Tools & Technologies

| Category | Tools & Technologies |
|---|---|
| Hardware | Raspberry Pi 5 (1 GB), MacBook Air, Bbox Wi-Fi 6 Router |
| Operating System | Raspberry Pi OS Lite (64-bit) |
| System Setup | Raspberry Pi Imager, NetworkManager, `nmcli` |
| DNS & Ad Blocking | Pi-hole, Quad9, StevenBlack Blocklist |
| VPN & Networking | WireGuard, `iptables`, `qrencode` |
| System Security | UFW, SSH, SSH Keys, unattended-upgrades |
| Network Monitoring | arp-scan, Cron, ntfy |
| Backup & Administration | `scp`, Pi-hole Teleporter, Linux command line |

## Objective

Build my own home server using a Raspberry Pi 5.

The main goals were:

- Block ads and trackers on my devices using Pi-hole.
- Set up a personal VPN to connect safely when I'm away from home.
- Improve server security with a firewall, automatic updates and SSH keys.
- Get alerts when an unknown device connects to my network.

**Environment:** My own home network and devices.

**Equipment:** Raspberry Pi 5 (1 GB), case, fan, heatsinks, 27 W power supply, 64 GB microSD card, MacBook Air and Bouygues Telecom Bbox Wi-Fi 6 router.

---

## 1. Raspberry Pi Setup

I assembled the Raspberry Pi inside its case and installed the fan.

I could only install one small heatsink because the others were too close to the fan.

I installed **Raspberry Pi OS Lite (64-bit)** using Raspberry Pi Imager.

I chose this version because it does not need a desktop interface.

I enabled SSH and connected the Raspberry Pi to my router with an Ethernet cable.

Then, I connected from my Mac and updated the system.

```bash
ssh <usuario>@<nombre-raspberry>.local
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

## 2. Pi-hole: Blocking Ads and Trackers

I installed Pi-hole to block ads and trackers using DNS.

```bash
curl -sSL https://install.pi-hole.net | bash
sudo pihole setpassword
```

**Configuration:**

- **DNS provider:** Quad9 (filtered, with DNSSEC).
- **Blocklist:** StevenBlack, with around 71,000 domains.
- **Devices:** Mac, iPhone and Samsung TV.

I manually changed the DNS settings on these devices to use my Raspberry Pi's IP address.

I did not change the DNS settings for the whole network.

### Problem: Router Page Not Working

After using Pi-hole, I could not open my router's settings page.

The router used a local name that Pi-hole did not know.

**Solution:**

I added a local DNS record in Pi-hole:

`mabbox.bytel.fr → 192.168.1.254`

I also cleared the DNS cache on my Mac.

After this change, I could open the router's settings page again.

## 3. Static IP Address

I wanted my Raspberry Pi to keep the same IP address.

I configured a static IP address using NetworkManager.

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method manual \
  ipv4.addresses 192.168.1.158/24 ipv4.gateway 192.168.1.254 \
  ipv4.dns "9.9.9.9 149.112.112.112"
sudo nmcli connection up "Wired connection 1"
```

I also reserved the same IP address on my router using the Raspberry Pi's MAC address.

**Result:** The Raspberry Pi uses `192.168.1.158`.

## 4. WireGuard VPN Setup

I installed WireGuard to access my home network when I'm away.

I configured it manually without using automatic installation tools.

```bash
sudo apt install -y wireguard qrencode iptables
```

### Creating Security Keys

I created separate keys for the server and my iPhone.

```bash
sudo sh -c 'umask 077; cd /etc/wireguard; wg genkey | tee servidor.key | wg pubkey > servidor.pub; wg genkey | tee iphone.key | wg pubkey > iphone.pub'
```

The private keys were protected so only root could read them.

### Server Configuration

File: `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <server private key>
PostUp = iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

[Peer]
# iPhone
PublicKey = <iPhone public key>
AllowedIPs = 10.8.0.2/32
```

I enabled IP forwarding and started WireGuard.

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl -p /etc/sysctl.d/99-wireguard.conf
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

### Router Port Forwarding

I opened one UDP port on my router:

- **External port:** 20000
- **Internal port:** 51820
- **Destination:** Raspberry Pi

My internet provider shares public IPv4 addresses between customers.

Because of this, my router only allowed external ports between 16384 and 24575.

### iPhone Configuration

I created a QR code using `qrencode` to add the VPN to my iPhone.

I configured:

`DNS = 10.8.0.1`

This allows my iPhone to use Pi-hole when I'm away from home.

I also enabled automatic VPN connection:

- When using mobile data.
- When connecting to Wi-Fi networks outside my home.

## 5. Server Security

I added several security settings to protect my Raspberry Pi.

### SSH Keys

I created an SSH key on my Mac.

```bash
ssh-keygen -t ed25519 -C "mac"
ssh-copy-id <usuario>@<nombre-raspberry>.local
```

I kept password login as a backup.

### Automatic Updates

I installed automatic security updates.

```bash
sudo apt install -y unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades
```

I checked that both settings were enabled with `1`.

### Firewall (UFW)

I configured the firewall to block incoming connections by default.

I only allowed the ports needed for my services.

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 53 comment 'Pi-hole DNS'
sudo ufw allow 80/tcp comment 'Panel Pi-hole'
sudo ufw allow 5353/udp comment 'mDNS (.local)'
sudo ufw allow 51820/udp comment 'WireGuard'
sudo ufw route allow in on wg0 out on eth0 comment 'VPN a internet'
sudo ufw enable
sudo ufw status verbose
```

### Backups

I saved a backup of my Pi-hole settings using **Settings → Teleporter**.

I also copied the `/etc/wireguard` configuration folder to my Mac using `scp`.

## 6. Unknown Device Alerts

I created a system to detect unknown devices on my network.

The system uses:

- **arp-scan:** Finds devices connected to the local network.
- **Cron:** Runs the script every five minutes.
- **ntfy:** Sends notifications to my phone.

The script compares the MAC addresses it finds with a list of known devices.

If it finds a new MAC address, it saves the information and sends an alert.

```bash
sudo apt install -y arp-scan cron
sudo arp-scan --localnet --quiet --plain
```

I configured the script to run every five minutes.

```bash
echo '*/5 * * * * root /usr/local/bin/alarma-intrusos.sh' | sudo tee /etc/cron.d/alarma-intrusos
```

**Result:** The system detected my Samsung TV when it connected for the first time.

It sent only one notification for the new device.

## 7. Network Inventory

I used arp-scan to check the devices on my network.

```bash
cd /tmp && sudo arp-scan --localnet
```

The scan showed IP addresses, MAC addresses and available manufacturer information.

I found three unknown devices with similar MAC address prefixes.

The manufacturer was not identified.

### How I Found Them

I disconnected devices one by one and repeated the scan.

I checked which MAC addresses disappeared.

**Result:** The three unknown devices were the three smart bulbs in my lamp.

---

## Results

### Pi-hole

Pi-hole worked on my Mac, iPhone and Samsung TV.

During a test on a news website, it blocked **38 out of 250 requests (15%)**.

### WireGuard VPN

The VPN worked using mobile data.

My iPhone appeared in Pi-hole with the IP address `10.8.0.2`.

Pi-hole continued blocking trackers when I was outside my home network.

### Server Security

The Raspberry Pi had:

- A static IP address.
- An active firewall.
- Automatic security updates.
- SSH key authentication.
- Configuration backups.

### Device Alerts

The alert system detected my Samsung TV when it first connected.

It sent one notification for the new device.

### Network Inventory

I identified all three unknown devices as smart bulbs.

### Unresolved Problem

WhatsApp did not connect when I used the VPN outside my home network.

Other applications worked correctly.

**Possible solutions to test:**

- Change the VPN MTU to `1280`.
- Remove `::/0` from the iPhone's `AllowedIPs` setting because the VPN only uses IPv4.

This problem is still not solved.

---

## Lessons Learned

### 1. Open Only the Ports You Need

I opened only one UDP port for WireGuard.

I did not use DMZ because it could expose more services on my Raspberry Pi to the internet.

### 2. Check Settings Before Making Changes

Before changing important settings, I learned to check the current configuration.

Useful commands include:

- `nmcli` — Check network settings.
- `sshd -t` — Check SSH configuration.
- `grep -v PrivateKey` — View configuration text without showing lines containing `PrivateKey`.

I also learned to keep an SSH session open until I had tested the new connection from another terminal.

### 3. Protect Private Keys

Private keys must not be shared.

I used `umask 077` to protect the key files.

I also avoided showing private keys when checking configuration files.

### 4. DNS Can Cause Connection Problems

After changing my DNS settings to Pi-hole, I could not open my router's administration page.

I solved the problem by adding a local DNS record and clearing my Mac's DNS cache.

This helped me understand how DNS works on a home network.

### 5. Understand Your Internet Provider

My provider uses shared public IPv4 addresses and gives customers specific port ranges.

I had to understand this limitation to configure WireGuard correctly.

### 6. MAC Addresses Do Not Always Identify Devices

A MAC address can help identify a device, but the manufacturer is not always available.

In my case, disconnecting devices one by one helped me find the three smart bulbs.

### 7. Smart Devices Need Extra Attention

Smart home devices can create security risks.

Using a separate guest network can help keep these devices away from other devices on the main network.

### 8. Network Changes Can Take Time

My Samsung TV did not appear in Pi-hole immediately.

I learned to wait and check the configuration before deciding that something was wrong.

---

## Conclusion

In this project, I built and configured my own Raspberry Pi home security server.

I used Pi-hole to block ads and trackers, WireGuard to connect to my home network, and UFW to control network connections.

I also created a system to detect unknown devices and send alerts to my phone.

This project helped me improve my practical skills in Linux, networking, DNS, VPNs and basic system security.

I also learned how to solve configuration problems and document my work step by step.
