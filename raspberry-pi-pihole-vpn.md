## Practice: Raspberry Pi 5 as a Home Security Server (Pi-hole + WireGuard VPN)

**Date:** 25/10/2026

**Objective:** Set up a small home server from scratch to block ads and trackers on my devices, provide my own VPN to use when I'm away from home, keep it secure (firewall, updates, key-based access), and notify me if an unknown device connects to my network.

**Permission obtained from:** My own network and devices.

**Equipment:** Raspberry Pi 5 (1 GB) kit with case, fan, heatsinks, 27 W power supply, and 64 GB microSD card · MacBook Air · Bouygues Telecom Bbox Wi-Fi 6 router.

---

### Steps and commands

**1. Assembly and operating system installation**
- Assembled the board inside the case with the fan. I only installed one small heatsink because the others interfered with the fan.
- Installed **Raspberry Pi OS Lite (64-bit)**, without a desktop environment, using Raspberry Pi Imager, with SSH enabled and a wired connection.
- Connected from my Mac and updated the system:
```bash
ssh <usuario>@<nombre-raspberry>.local
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

**2. Pi-hole (blocking ads and trackers through DNS)**
```bash
curl -sSL https://install.pi-hole.net | bash
sudo pihole setpassword
```
- Upstream DNS: **Quad9 (filtered, DNSSEC)**. Blocklist: **StevenBlack**, around 71,000 domains.
- I use it **only on the devices I choose** (Mac, iPhone, TV), manually changing their DNS settings to the Raspberry Pi's IP address.
- Added a **local DNS record** in Pi-hole so the router's administration page would still work (`mabbox.bytel.fr → 192.168.1.254`).

**3. Static IP address for the Raspberry Pi**
- On the Raspberry Pi itself (NetworkManager):
```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method manual \
  ipv4.addresses 192.168.1.158/24 ipv4.gateway 192.168.1.254 \
  ipv4.dns "9.9.9.9 149.112.112.112"
sudo nmcli connection up "Wired connection 1"
```
- On the router: **static IP reservation** (DHCP) for the Raspberry Pi's MAC address.

**4. WireGuard VPN, configured manually**
- Used WireGuard without automated installers (PiVPN stopped being maintained in 2024).
```bash
sudo apt install -y wireguard qrencode iptables
# Key pairs for the server and phone (readable only by root)
sudo sh -c 'umask 077; cd /etc/wireguard; wg genkey | tee servidor.key | wg pubkey > servidor.pub; wg genkey | tee iphone.key | wg pubkey > iphone.pub'
```
- `/etc/wireguard/wg0.conf` (server):
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
```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl -p /etc/sysctl.d/99-wireguard.conf
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```
- **Port forwarding on the router:** UDP, external port `20000` → port `51820` on the Raspberry Pi. The router only allowed ports between 16384 and 24575 because the ISP **shares public IPv4 addresses among multiple customers** and assigns specific port ranges.
- Generated the phone's configuration as a **QR code** (`qrencode`) with `DNS = 10.8.0.1`, so I can also use Pi-hole when I'm away from home.
- In the WireGuard app on my iPhone: **automatic activation** on mobile data and on any Wi-Fi network except my home network.

**5. Security hardening**
- **SSH key authentication** from my Mac (I also kept password authentication as a backup):
```bash
ssh-keygen -t ed25519 -C "mac"
ssh-copy-id <usuario>@<nombre-raspberry>.local
```
- **Automatic security updates:**
```bash
sudo apt install -y unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades   # both lines must end in "1"
```
- **Firewall (ufw):** everything blocked by default except what's necessary:
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
- **Backups:** exported Pi-hole settings (Settings → Teleporter) and copied `/etc/wireguard` to my Mac using `scp`.

**6. Intruder alert system**
- A script that runs an `arp-scan` every 5 minutes (cron), compares MAC addresses against a list of known devices, and, if a new one appears, logs it and sends a notification to my phone through **ntfy**.
```bash
sudo apt install -y arp-scan cron
sudo arp-scan --localnet --quiet --plain
# Run every 5 minutes
echo '*/5 * * * * root /usr/local/bin/alarma-intrusos.sh' | sudo tee /etc/cron.d/alarma-intrusos
```

**7. Network inventory**
```bash
cd /tmp && sudo arp-scan --localnet   # also shows the manufacturer of each MAC address
```
- Identified the unknown devices (three with the same MAC address prefix and no recognized manufacturer) **by unplugging devices one by one** and checking which one disappeared.

---

### Results
- Pi-hole working on my Mac, iPhone, and Samsung TV. During the first test on a news website, it blocked **15%** of requests (38 out of 250).
- WireGuard VPN working over mobile data. My iPhone appears in Pi-hole as `10.8.0.2` and continues blocking trackers outside my home network.
- Raspberry Pi with a static IP address (configured on the system itself and reserved on the router), active firewall, automatic updates, SSH key authentication, and backups.
- The alert system successfully detected the TV the first time it connected and only sends one notification per new device.
- Complete network inventory: the three mystery devices turned out to be **the three smart bulbs in a lamp**.
- **Unresolved issue:** with the VPN enabled outside my home network, WhatsApp won't connect (all other apps work). Possible solutions to test: lowering the tunnel's MTU to 1280 or removing `::/0` from the phone's AllowedIPs (the tunnel only carries IPv4 traffic).

### Lessons learned
- **Only open what's absolutely necessary.** For the VPN, I forwarded a single UDP port instead of using DMZ, which would have exposed the entire Raspberry Pi.
- **Before creating or changing a configuration, check it first and then apply it** (`nmcli … show`, `sshd -t`, `grep -v PrivateKey`), and always have a way to undo the change. For example, don't close an SSH session until you've tested access from another terminal window.
- **Never expose private keys.** Files are created with `umask 077`, and the private key line is hidden when reviewing a configuration.
- **DNS explains many strange issues.** After switching to Pi-hole, the router's administration page stopped loading because only the router knew its hostname. I fixed it by adding a local DNS record and flushing the Mac's DNS cache.
- **You need to understand your ISP's limitations:** shared public IPv4 addresses with assigned port ranges, and router settings hidden in less obvious menus.
- **The MAC address manufacturer can help identify a device**, but it isn't always listed in the database. Sometimes the most reliable method is physical: unplug a device and see what disappears.
- **Cheap smart home devices are a weak point in a home network.** Ideally, they should be isolated on a guest network.
- **Network changes can take time to take effect.** The TV didn't appear in Pi-hole straight away, so you shouldn't assume a configuration has failed too quickly.