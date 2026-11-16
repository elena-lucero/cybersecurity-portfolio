<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:0D1117,100:3EC6D6&height=170&section=header&text=Metasploitable2%20Lab&fontSize=42&fontColor=F0F6FC&fontAlignY=38&desc=LAB%2001%20%C2%B7%20Isolated%20practice%20network&descSize=16&descAlignY=60&descColor=F0F6FC" width="100%" alt="Metasploitable2 Lab">
</p>

<p align="center">
  <a href="https://github.com/elena-lucero/cybersecurity-portfolio"><img src="https://img.shields.io/badge/%E2%86%90_Back_to_portfolio-0D1117?style=for-the-badge&logo=github&logoColor=3EC6D6" alt="Back to portfolio"></a>
</p>

<p align="center"><code>UTM</code> <code>Kali Linux</code> <code>Metasploitable2</code> <code>Host-only network</code></p>

<img src="https://capsule-render.vercel.app/api?type=rect&color=0:0D1117,50:3EC6D6,100:0D1117&height=2" width="100%" alt="">

## `$ cat overview.txt`

A safe space to practise: an attacker machine and an intentionally vulnerable target,
running as virtual machines in **UTM** on my Mac and connected only to each other.

| Machine | Role |
|---|---|
| **Kali LAB** | Dedicated copy of Kali Linux used only for this lab |
| **Metasploitable2** | Intentionally vulnerable Linux machine, built for security practice |

<img src="https://capsule-render.vercel.app/api?type=rect&color=0:0D1117,50:3EC6D6,100:0D1117&height=2" width="100%" alt="">

## `$ cat architecture.txt`

```mermaid
flowchart LR
  subgraph UTM["UTM · isolated host-only network"]
    K["Kali LAB"] --- M["Metasploitable2"]
  end
  I["Internet / home network"] -. "no access" .- UTM
  classDef node fill:#0D1117,stroke:#3EC6D6,stroke-width:1px,color:#F0F6FC
  class K,M,I node
  style UTM fill:transparent,stroke:#3EC6D6,stroke-dasharray:4 4,color:#3EC6D6
```

<img src="https://capsule-render.vercel.app/api?type=rect&color=0:0D1117,50:3EC6D6,100:0D1117&height=2" width="100%" alt="">

## `$ cat why_isolated.txt`

Metasploitable2 is vulnerable on purpose, so it must never be reachable from the internet
or from my home network. Using a **host-only network** in UTM keeps both machines
in their own closed space: they can see each other, and nothing else.

<img src="https://capsule-render.vercel.app/api?type=rect&color=0:0D1117,50:3EC6D6,100:0D1117&height=2" width="100%" alt="">

## `$ cat setup.txt`

1. Created a dedicated copy of Kali Linux in UTM, **Kali LAB**, so my main Kali stays separate.
2. Added **Metasploitable2** as a second virtual machine in UTM.
3. Connected both machines to the same **host-only** network.
4. Checked that they can reach each other.

<img src="https://capsule-render.vercel.app/api?type=rect&color=0:0D1117,50:3EC6D6,100:0D1117&height=2" width="100%" alt="">

## `$ ping metasploitable2`

Connectivity between both machines confirmed with `ping` from Kali LAB:

```bash
ping -c 4 <METASPLOITABLE2_IP>
```

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:3EC6D6,100:0D1117&height=110&section=footer" width="100%" alt="footer">
</p>
