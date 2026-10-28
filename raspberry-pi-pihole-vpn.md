## Práctica: Raspberry Pi 5 como servidor de seguridad para casa (Pi-hole + VPN WireGuard)

**Fecha:** 25/10/2026

**Objetivo:** Montar desde cero un pequeño servidor casero que bloquee anuncios y rastreadores en mis aparatos, me dé una VPN propia para usar fuera de casa, esté protegido (cortafuegos, actualizaciones, acceso por llave) y me avise si se conecta un aparato desconocido a mi red.

**Permiso obtenido de:** Red y aparatos propios. Mi pareja, con quien comparto la red, está informado y de acuerdo.

**Material:** Raspberry Pi 5 (1 GB) en kit con caja, ventilador, disipadores, cargador de 27 W y microSD de 64 GB · MacBook Air · router Bbox Wi-Fi 6 de Bouygues Telecom.

---

### Pasos y comandos

**1. Montaje e instalación del sistema**
- Montaje de la placa en la caja con ventilador. Solo dejé un disipador pequeño, porque los demás chocaban con el ventilador.
- Instalación de **Raspberry Pi OS Lite (64-bit)**, sin escritorio, con Raspberry Pi Imager, SSH activado y conexión por cable.
- Conexión desde el Mac y actualización:
```bash
ssh <usuario>@<nombre-raspberry>.local
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

**2. Pi-hole (bloqueo de anuncios y rastreadores por DNS)**
```bash
curl -sSL https://install.pi-hole.net | bash
sudo pihole setpassword
```
- DNS de subida: **Quad9 (filtered, DNSSEC)**. Lista de bloqueo: **StevenBlack**, unos 71.000 dominios.
- Lo uso **solo en los aparatos que elijo** (Mac, iPhone, tele), cambiando su DNS a mano a la IP de la Raspberry.
- Añadí un **registro DNS local** en Pi-hole para que siga funcionando la página de administración del router (`mabbox.bytel.fr → 192.168.1.254`).

**3. IP fija para la Raspberry**
- En la propia Raspberry (NetworkManager):
```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method manual \
  ipv4.addresses 192.168.1.158/24 ipv4.gateway 192.168.1.254 \
  ipv4.dns "9.9.9.9 149.112.112.112"
sudo nmcli connection up "Wired connection 1"
```
- En el router: **reserva de IP estática** (DHCP) para la MAC de la Raspberry.

**4. VPN con WireGuard, configurada a mano**
- Usé WireGuard sin instaladores automáticos (PiVPN dejó de mantenerse en 2024).
```bash
sudo apt install -y wireguard qrencode iptables
# Pareja de llaves para el servidor y para el móvil (solo legibles por root)
sudo sh -c 'umask 077; cd /etc/wireguard; wg genkey | tee servidor.key | wg pubkey > servidor.pub; wg genkey | tee iphone.key | wg pubkey > iphone.pub'
```
- `/etc/wireguard/wg0.conf` (servidor):
```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <llave privada del servidor>
PostUp = iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

[Peer]
# iPhone
PublicKey = <llave pública del iPhone>
AllowedIPs = 10.8.0.2/32
```
```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl -p /etc/sysctl.d/99-wireguard.conf
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```
- **Redirección de puerto en el router:** UDP, puerto externo `20000` → puerto `51820` de la Raspberry. El router solo permitía puertos entre 16384 y 24575, porque la compañía **comparte la IPv4 pública entre varios clientes** y reparte rangos de puertos.
- Configuración del móvil generada como **código QR** (`qrencode`) con `DNS = 10.8.0.1`, para usar Pi-hole también fuera de casa.
- En la app WireGuard del iPhone: **activación automática** con datos móviles y en cualquier wifi excepto la de casa.

**5. Refuerzo de seguridad**
- **SSH con llave** desde el Mac (mantengo también la contraseña como respaldo):
```bash
ssh-keygen -t ed25519 -C "mac"
ssh-copy-id <usuario>@<nombre-raspberry>.local
```
- **Actualizaciones de seguridad automáticas:**
```bash
sudo apt install -y unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades   # las dos líneas deben acabar en "1"
```
- **Cortafuegos (ufw):** todo cerrado por defecto salvo lo necesario:
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
- **Copias de seguridad:** exportación de Pi-hole (Settings → Teleporter) y copia de `/etc/wireguard` llevada al Mac con `scp`.

**6. Alarma de intrusos**
- Script que cada 5 minutos (cron) hace un `arp-scan` de la red, compara las MAC con una lista de aparatos conocidos y, si aparece uno nuevo, lo apunta y me manda un aviso al móvil con **ntfy**.
```bash
sudo apt install -y arp-scan cron
sudo arp-scan --localnet --quiet --plain
# Despertador cada 5 minutos
echo '*/5 * * * * root /usr/local/bin/alarma-intrusos.sh' | sudo tee /etc/cron.d/alarma-intrusos
```

**7. Inventario de la red**
```bash
cd /tmp && sudo arp-scan --localnet   # muestra también el fabricante de cada MAC
```
- Identifiqué los aparatos desconocidos (tres con el mismo prefijo de MAC, sin fabricante conocido) **desenchufando aparatos uno a uno** y viendo cuál desaparecía.

---

### Resultado
- Pi-hole funcionando en Mac, iPhone y tele Samsung. En la primera prueba con una web de noticias bloqueó el **15 %** de las peticiones (38 de 250).
- VPN WireGuard funcionando con datos móviles. El iPhone aparece en Pi-hole como `10.8.0.2` y sigue bloqueando rastreadores fuera de casa.
- Raspberry con IP fija (en el propio sistema y reservada en el router), cortafuegos activo, actualizaciones automáticas, acceso SSH por llave y copias de seguridad.
- La alarma detectó correctamente la tele la primera vez que se conectó, y avisa de cada aparato nuevo solo una vez.
- Inventario de red completo: los tres aparatos misteriosos eran **las tres bombillas inteligentes de una lámpara**.
- **Problema abierto:** con la VPN encendida fuera de casa, WhatsApp no conecta (el resto de apps sí). Hipótesis a probar: bajar la MTU del túnel a 1280 o quitar `::/0` de las IPs permitidas del móvil (el túnel solo lleva IPv4).

### Lección aprendida
- **Solo hay que abrir lo imprescindible.** Para la VPN usé redirección de un único puerto UDP y no la DMZ, que habría expuesto la Raspberry entera.
- **Para crear o cambiar una configuración, primero se comprueba y luego se aplica** (`nmcli … show`, `sshd -t`, `grep -v PrivateKey`), y hay que tener una vía para deshacer el cambio. Por ejemplo, no cerrar la sesión SSH hasta probar el acceso desde otra ventana.
- **Las llaves privadas nunca se enseñan.** Los archivos se crean con `umask 077`, y al revisar una configuración se oculta la línea de la llave.
- **El DNS explica muchos fallos raros.** Tras usar Pi-hole, la página del router dejó de cargar porque su nombre solo lo conocía el router. Se arregló con un registro DNS local y vaciando la caché DNS del Mac.
- **Hay que conocer las limitaciones del proveedor:** IPv4 compartida con un rango de puertos asignado, y opciones del router que están en menús poco evidentes.
- **El fabricante de la MAC delata el aparato**, pero no siempre está en la base de datos. A veces la técnica más fiable es la física: quitar un aparato y ver qué desaparece.
- **Los aparatos inteligentes baratos son el punto débil de una red casera.** Lo ideal sería aislarlos en una red de invitados.
- **Un cambio de red puede tardar en aplicarse.** La tele no apareció en Pi-hole hasta pasado un rato, y no hay que descartar una configuración demasiado pronto.
