<div align="center">

# 🌐 Static NAT: One Private Host, One Public Address

### Network Address Translation on a Cisco Router · Packet Tracer

![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Packet Tracer](https://img.shields.io/badge/Packet%20Tracer-Lab-049fd9?style=for-the-badge)
![NAT](https://img.shields.io/badge/NAT-Static-red?style=for-the-badge)
![Mapping](https://img.shields.io/badge/Mapping-One--to--One-orange?style=for-the-badge)
![Debug](https://img.shields.io/badge/Verified%20with-debug%20ip%20nat-purple?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Verified-success?style=for-the-badge)

*Private addresses stay private. The router speaks for them.*

</div>

---

## 📑 Table of Contents
1. [Overview](#-overview)
2. [Topology](#-topology)
3. [Addressing Table](#-addressing-table)
4. [NAT Theory](#-nat-theory)
5. [NAT Terminology in This Lab](#-nat-terminology-in-this-lab)
6. [Configuration](#-configuration)
7. [Verification: Stage by Stage](#-verification-stage-by-stage)
8. [Reading the Debug Output](#-reading-the-debug-output)
9. [Before vs After](#-before-vs-after)
10. [How Static NAT Behaves](#-how-static-nat-behaves)
11. [Key Takeaways](#-key-takeaways)
12. [Troubleshooting Notes](#-troubleshooting-notes)
13. [Next Labs](#-next-labs)
14. [Author](#-author)

---

## 🎯 Overview

Private (RFC 1918) addresses such as `192.168.1.0/24` cannot be routed on the Internet. **NAT (Network Address Translation)** lets a router replace them with a usable address.

**Static NAT** is the simplest form: a **permanent, one-to-one** mapping between one private (inside local) address and one public (inside global) address.

| Item | Detail |
|---|---|
| NAT type | **Static NAT** (1 static, 0 dynamic) |
| NAT router | Router0, hostname `NAT-ROUTER` |
| Inside interface | GigabitEthernet0/0 (`ip nat inside`) |
| Outside interface | Serial0/0/0 (`ip nat outside`) |
| Translated host | 192.168.1.4 (PC0) |
| Public (inside global) address | **1.1.1.40** |
| ISP router | Router1, hostname `ISP-ROUTER` (1.1.1.2) |
| Proof | `debug ip nat` on both routers, plus the translation table and statistics |

---

## 🗺 Topology

```
   ┌──────────────── Private Network ────────────────┐        ┌──── ISP ────┐
   │                                                  │        │             │
   │   PC0 192.168.1.4   PC1 192.168.1.3   PC2 192.168.1.2     │             │
   │          \               |               /      │        │             │
   │           \              |              /       │        │             │
   │             ┌─────────────────────┐             │        │             │
   │             │  Switch0 (2950-24)  │             │        │             │
   │             └──────────┬──────────┘             │        │             │
   │                        │                        │        │             │
   │               Gi0/0 192.168.1.1                 │        │             │
   │               ip nat INSIDE                     │        │             │
   │                  ┌─────┴─────┐  Se0/0/0 1.1.1.1 ━━━━━━━━━⚡━━━ 1.1.1.2 ┌────────┐
   │                  │  Router0  │  ip nat OUTSIDE  │        │             │ Router1│
   │                  │NAT-ROUTER │                  │        │             │  ISP   │
   │                  └───────────┘                  │        │             └────────┘
   └──────────────────────────────────────────────────┘        └─────────────┘

   Translation:   192.168.1.4  ◄══════ static ══════►  1.1.1.40
```

---

## 📋 Addressing Table

| Device | Interface | IP Address | Role |
|---|---|---|---|
| Router0 (NAT-ROUTER) | GigabitEthernet0/0 | 192.168.1.1/24 | Private gateway, **NAT inside** |
| Router0 (NAT-ROUTER) | Serial0/0/0 | 1.1.1.1 | WAN link, **NAT outside** |
| Router1 (ISP-ROUTER) | Serial | 1.1.1.2 | ISP side of the WAN link |
| PC0 | NIC | **192.168.1.4/24** | Host with the **static NAT entry** |
| PC1 | NIC | 192.168.1.3/24 | Private host (no NAT entry) |
| PC2 | NIC | 192.168.1.2/24 | Private host (no NAT entry) |
| (NAT mapping) | n/a | **1.1.1.40** | Public address for PC0 |

> The WAN link sits in the `1.0.0.0` network shown in the diagram. Because 1.1.1.40 is in the same network as the ISP link, the ISP can return traffic to it without extra routes.

---

## 📚 NAT Theory

| NAT type | Mapping | Typical use |
|---|---|---|
| **Static NAT (this lab)** | **One private ↔ one public**, permanent | Servers that need a fixed public address |
| Dynamic NAT | Private → pool of public addresses, assigned on demand | Many hosts, a pool of public IPs |
| PAT / NAT overload | Many private → **one** public IP, separated by port numbers | Home and office Internet access |

### Why NAT?

| Reason | Explanation |
|---|---|
| Private addresses are not routable | ISPs and the Internet do not carry 192.168.x.x |
| Saves public addresses | Inside hosts do not each need their own |
| Hides the internal layout | Outsiders only see the translated address |

---

## 🏷 NAT Terminology in This Lab

| Term | Meaning | Value in this lab |
|---|---|---|
| **Inside local** | The inside host's private address, as seen **inside** | 192.168.1.4 |
| **Inside global** | The inside host's address as seen from the **outside** | 1.1.1.40 |
| **Outside local** | The outside host's address as seen from the inside | 1.1.1.2 (ISP, untranslated) |
| **Outside global** | The outside host's real address | 1.1.1.2 |
| **Inside interface** | Faces the private network | Gi0/0 |
| **Outside interface** | Faces the public side | Se0/0/0 |

```
 Inside local ──────► [ NAT-ROUTER ] ──────► Inside global
 192.168.1.4                                   1.1.1.40
 (what PC0 really is)                    (what the ISP sees)
```

---

## ⚙️ Configuration

### NAT-ROUTER (Router0)

```
NAT-ROUTER(config)# ip nat inside source static 192.168.1.4 1.1.1.40

NAT-ROUTER(config)# interface gigabitEthernet 0/0
NAT-ROUTER(config-if)# ip nat inside
NAT-ROUTER(config-if)# exit

NAT-ROUTER(config)# interface serial 0/0/0
NAT-ROUTER(config-if)# ip nat outside
NAT-ROUTER(config-if)# end
```

### Basic connectivity
> Replace this block with the routing you actually used.

```
NAT-ROUTER(config)# ip route 0.0.0.0 0.0.0.0 1.1.1.2
```

### Debugging (used for the proof)

```
NAT-ROUTER# debug ip nat       ! on the NAT router
ISP-ROUTER# debug ip icmp      ! ICMP replies seen from the ISP side
ISP-ROUTER# debug ip nat       ! enabled on the ISP router as well
```
> Turn debugging off afterwards with `undebug all`.

### Config breakdown

| Command | Purpose |
|---|---|
| `ip nat inside source static 192.168.1.4 1.1.1.40` | Permanent mapping: inside local 192.168.1.4 ↔ inside global 1.1.1.40 |
| `interface gigabitEthernet 0/0` + `ip nat inside` | Marks the private side |
| `interface serial 0/0/0` + `ip nat outside` | Marks the public side |

> Without the inside/outside markings, the static entry exists but **nothing is translated**.

---

## ✅ Verification: Stage by Stage

### 🔴 Stage 1: BEFORE NAT: the ISP sees the private hosts

On the ISP router, ICMP debugging shows the pings arriving and the replies being generated:

```
ISP-ROUTER#
ICMP: echo reply sent, src 1.1.1.2, dst 192.168.1.2
ICMP: echo reply sent, src 1.1.1.2, dst 192.168.1.2
ICMP: echo reply sent, src 1.1.1.2, dst 192.168.1.2
ICMP: echo reply sent, src 1.1.1.2, dst 192.168.1.2

ICMP: echo reply sent, src 1.1.1.2, dst 192.168.1.3
ICMP: echo reply sent, src 1.1.1.2, dst 192.168.1.3
ICMP: echo reply sent, src 1.1.1.2, dst 192.168.1.3
ICMP: echo reply sent, src 1.1.1.2, dst 192.168.1.3
```

**Annotations**
- The ISP router knows the **real private addresses** (192.168.1.2 and 192.168.1.3) of the inside hosts.
- Private addresses are **not routable on the Internet**, so on a real ISP these replies could never find their way back.
- The internal addressing plan is **exposed** to the outside world.

---

### 🔧 Stage 2: Apply Static NAT

```
NAT-ROUTER(config)# ip nat inside source static 192.168.1.4 1.1.1.40
NAT-ROUTER(config)# interface gigabitEthernet 0/0
NAT-ROUTER(config-if)# ip nat inside
NAT-ROUTER(config)# interface serial 0/0/0
NAT-ROUTER(config-if)# ip nat outside
```

---

### 🟢 Stage 3: AFTER NAT: debug on the NAT router

PC0 (192.168.1.4) pings the ISP router (1.1.1.2), with `debug ip nat` on:

```
NAT-ROUTER# debug ip nat
IP NAT debugging is on

NAT:  s=192.168.1.4->1.1.1.40, d=1.1.1.2 [6]
NAT*: s=1.1.1.2, d=1.1.1.40->192.168.1.4 [2]
NAT:  s=192.168.1.4->1.1.1.40, d=1.1.1.2 [7]
NAT*: s=1.1.1.2, d=1.1.1.40->192.168.1.4 [3]
NAT:  s=192.168.1.4->1.1.1.40, d=1.1.1.2 [8]
NAT*: s=1.1.1.2, d=1.1.1.40->192.168.1.4 [4]
NAT:  s=192.168.1.4->1.1.1.40, d=1.1.1.2 [9]
NAT*: s=1.1.1.2, d=1.1.1.40->192.168.1.4 [5]
```

**Annotations**
- **Outbound** (`NAT:`): the **source** 192.168.1.4 is rewritten to **1.1.1.40** before the packet leaves for the ISP.
- **Inbound** (`NAT*:`): the reply's **destination** 1.1.1.40 is rewritten back to **192.168.1.4**, so the reply reaches PC0.
- Four pings produce four outbound and four inbound translations, which gives **8 hits** in the statistics later.

---

### 🟢 Stage 4: AFTER NAT: the ISP's view

```
ISP-ROUTER# debug ip nat
IP NAT debugging is on

ICMP: echo reply sent, src 1.1.1.2, dst 1.1.1.40
ICMP: echo reply sent, src 1.1.1.2, dst 1.1.1.40
ICMP: echo reply sent, src 1.1.1.2, dst 1.1.1.40
ICMP: echo reply sent, src 1.1.1.2, dst 1.1.1.40
```

**Annotations**
- The ISP now replies to **1.1.1.40**. It **never sees** 192.168.1.4.
- No NAT lines appear on the ISP router, because **all translation happens on NAT-ROUTER**. The ISP only knows the public address.

---

### 📋 Stage 5: Translation table

```
NAT-ROUTER# show ip nat translations
Pro  Inside global     Inside local       Outside local      Outside global
---  1.1.1.40          192.168.1.4        ---                ---
```

**Annotations**
- One entry: **inside global 1.1.1.40 ↔ inside local 192.168.1.4**.
- The outside columns are `---` because this is a simple static entry, with no per-session detail.
- The entry exists **before** any traffic and **stays** after traffic stops. That is what makes it static.

---

### 📊 Stage 6: NAT statistics

```
NAT-ROUTER# show ip nat statistics
Total translations: 1 (1 static, 0 dynamic, 0 extended)
Outside Interfaces: Serial0/0/0
Inside Interfaces: GigabitEthernet0/0
Hits: 8  Misses: 28
Expired translations: 8
Dynamic mappings:
```

| Field | Value | Meaning |
|---|---|---|
| Total translations | 1 (1 static) | The single static mapping |
| Outside Interfaces | Serial0/0/0 | Marked `ip nat outside` ✅ |
| Inside Interfaces | GigabitEthernet0/0 | Marked `ip nat inside` ✅ |
| **Hits** | **8** | Packets translated: 4 requests + 4 replies |
| Misses | 28 | Lookups that found no translation. Expected, because only one host has an entry |
| Expired translations | 8 | A counter of earlier timed-out entries. The static entry itself never expires |
| Dynamic mappings | (empty) | No dynamic NAT configured |

---

## 🔎 Reading the Debug Output

```
NAT:  s=192.168.1.4->1.1.1.40, d=1.1.1.2 [6]
       │        │        │        │       └─ IP identification of the packet
       │        │        │        └─ destination (unchanged)
       │        │        └─ source AFTER translation (inside global)
       │        └─ source BEFORE translation (inside local)
       └─ no asterisk: translated in the normal (process-switched) path

NAT*: s=1.1.1.2, d=1.1.1.40->192.168.1.4 [2]
 │                  │          └─ destination AFTER translation (back to the private host)
 │                  └─ destination BEFORE translation (the public address)
 └─ asterisk: translated in the fast-switching path
```

| Direction | What changes | Debug shows |
|---|---|---|
| Inside → Outside | **Source** address | `s=192.168.1.4->1.1.1.40` |
| Outside → Inside | **Destination** address | `d=1.1.1.40->192.168.1.4` |

---

## ⚖️ Before vs After

| | Before NAT | After NAT |
|---|---|---|
| What the ISP sees | Real private sources: 192.168.1.2 and 192.168.1.3 | Only the public address 1.1.1.40 |
| Internal addressing | ❌ Exposed | ✅ Hidden |
| Routable on the Internet? | ❌ No (RFC 1918) | ✅ Yes (public address) |
| Translation table | Empty | 1 static entry |
| NAT router debug | Nothing | `s=192.168.1.4->1.1.1.40` and `d=1.1.1.40->192.168.1.4` |

---

## 🧠 How Static NAT Behaves

| Behaviour | Explanation |
|---|---|
| **One-to-one** | Exactly one private host maps to exactly one public address |
| **Permanent** | The entry is created by the command, not by traffic, and never times out |
| **Bidirectional** | The outside can start a connection **to 1.1.1.40**, and the router forwards it to 192.168.1.4 |
| **Per host** | Only 192.168.1.4 has an entry. PC1 and PC2 are not translated by this rule |
| **Needs inside + outside** | Both interfaces must be marked, or nothing is translated |

> The bidirectional behaviour is why Static NAT is the standard choice for **servers**: the public address stays the same and outside users can always reach it.

---

## 💡 Key Takeaways

| # | Takeaway |
|---|---|
| 1 | NAT replaces private addresses with public ones at the router |
| 2 | **Static NAT** is a permanent **one-to-one** mapping |
| 3 | You must mark an **inside** and an **outside** interface |
| 4 | Outbound traffic gets its **source** translated, and returning traffic gets its **destination** translated |
| 5 | Inside local = private, inside global = what the outside sees |
| 6 | Static entries never expire, which is ideal for servers |
| 7 | `debug ip nat`, `show ip nat translations` and `show ip nat statistics` prove it works |
| 8 | The ISP never sees the private address, so the internal network stays hidden |

---

## 🛠 Troubleshooting Notes

| Symptom | Check |
|---|---|
| Nothing is translated | Are both `ip nat inside` and `ip nat outside` set? Are they on the right interfaces? |
| Inside and outside swapped | Inside = private LAN side, outside = ISP side |
| Translation table empty | Is the static command typed correctly (private IP first, public IP second)? |
| Ping translated but no reply | Does the ISP have a route to the public address? Is the return path OK? |
| Hits stay at 0 | Is the traffic really crossing the NAT interfaces? |
| Debug floods the screen | Run `undebug all` |
| Clear dynamic entries | `clear ip nat translation *` (static entries stay) |

---

## 🚀 Next Labs

- [x] Static NAT: one private host to one public address ✅
- [ ] Static NAT for a server with inbound access
- [ ] Dynamic NAT with a pool of public addresses
- [ ] PAT / NAT overload (many hosts, one public IP)
- [ ] NAT with an ACL to choose which hosts are translated

---

## 👨‍💻 Author

**Maaz Khan**
CCNA-certified Network & NOC Engineer · Lower Dir, KPK, Pakistan

[![LinkedIn](https://img.shields.io/badge/LinkedIn-maazkhanms-0A66C2?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/maazkhanms)
[![GitHub](https://img.shields.io/badge/GitHub-maazkhanms-181717?style=for-the-badge&logo=github)](https://github.com/maazkhanms)

⭐ If this lab helped you, give the repo a star!
