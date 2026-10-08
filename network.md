# 🌐 Network

Overview of the network design of the homelab.

## 📑 Index

- [Design philosophy](#-design-philosophy)
- [Topology](#-topology)
- [Edge & NAT](#-edge--nat)
- [Segmentation](#-segmentation)
- [WiFi](#-wifi)
- [Firewall](#-firewall)
- [DHCP & DNS](#-dhcp--dns)

---

## 🧠 Design philosophy

The network is built around a few simple principles:

- **Segmentation by trust level**, not by convenience.
- **One router to rule them all**: a single OpenWRT device handles
  routing, firewalling, DHCP, and VLANs.
- **Delegate NAT to the edge**, keeping the internal router focused
  on segmentation and policy.
- **Treat WiFi and wired the same way**: same segments, same rules.
- **Default deny** between segments, with granular exceptions.

---

## 🗺️ Topology

```mermaid
flowchart TB
    Internet(("🌍 Internet"))
    ISP["📡 ISP Router<br/><i>NAT</i>"]
    DMZ{{"🔗 P2P / DMZ Link"}}
    OWRT["🔥 OpenWRT Router"]

    Internet --> ISP --> DMZ --> OWRT

    subgraph LAN["🏠 Segmented LAN"]
        direction TB
        VLAN_P["💻 Personal VLAN"]
        VLAN_I["📱 IoT VLAN"]
        WIFI_P["📶 SSID for Home"]
        WIFI_I["📶 SSID for IoT"]

        VLAN_P --> WIFI_P
        VLAN_I --> WIFI_I
    end

    OWRT --> VLAN_P
    OWRT --> VLAN_I

    classDef internet fill:#1e3a8a,stroke:#60a5fa,color:#fff
    classDef isp fill:#7c2d12,stroke:#fb923c,color:#fff
    classDef dmz fill:#4c1d95,stroke:#a78bfa,color:#fff
    classDef router fill:#14532d,stroke:#4ade80,color:#fff
    classDef vlanP fill:#0c4a6e,stroke:#38bdf8,color:#fff
    classDef vlanI fill:#713f12,stroke:#facc15,color:#fff
    classDef device fill:#1f2937,stroke:#9ca3af,color:#fff

    class Internet internet
    class ISP isp
    class DMZ dmz
    class OWRT router
    class VLAN_P vlanP
    class VLAN_I vlanI
    class WIFI_P,WIFI_I device
```

---

## 🔗 Edge & NAT

- **NAT is delegated to the ISP router**, which sits at the edge.
- The OpenWRT router is connected through a **dedicated point-to-point
  link** that acts as a **DMZ / transit segment**.
- This keeps the internal router focused on segmentation, firewalling
  and DHCP — not on managing the public boundary.
- The OpenWRT router **routes everything** from the LAN out through
  this transit link.

```mermaid
flowchart LR
    LAN["🏠 LAN"] --> OWRT["🔥 OpenWRT"]
    OWRT --> DMZ["🔗 P2P / DMZ"]
    DMZ --> ISP["📡 ISP Router<br/><i>NAT</i>"]
    ISP --> NET(("🌍 Internet"))

    classDef lan fill:#0c4a6e,stroke:#38bdf8,color:#fff
    classDef router fill:#14532d,stroke:#4ade80,color:#fff
    classDef dmz fill:#4c1d95,stroke:#a78bfa,color:#fff
    classDef isp fill:#7c2d12,stroke:#fb923c,color:#fff
    classDef net fill:#1e3a8a,stroke:#60a5fa,color:#fff

    class LAN lan
    class OWRT router
    class DMZ dmz
    class ISP isp
    class NET net
```

---

## 🧩 Segmentation

Two internal segments, separated by trust level:

| Segment       | Purpose                                | DHCP     |
|---------------|----------------------------------------|----------|
| Personal VLAN | Trusted devices (laptops, phones, etc.)| OpenWRT  |
| IoT VLAN      | Untrusted / smart devices              | OpenWRT  |

Both segments are served by the **same router** (OpenWRT), but they
are isolated by firewall policy.

---

## 📶 WiFi

- Each segment has its **own SSID**.
- WiFi traffic is tagged into the corresponding VLAN (802.1Q).
- Wired and wireless devices share the same policies per segment —
  the transport doesn't change the trust level.

---

## 🛡️ Firewall

Granular, **per-VLAN** firewall rules on the OpenWRT router:

```mermaid
flowchart LR
    P["💻 Personal"]
    I["📱 IoT"]
    W["🌍 Internet"]

    P -->|allowed| W
    I -->|allowed| W
    P -.->|restricted| I
    I -.-x|blocked| P

    classDef p fill:#0c4a6e,stroke:#38bdf8,color:#fff
    classDef i fill:#713f12,stroke:#facc15,color:#fff
    classDef w fill:#1e3a8a,stroke:#60a5fa,color:#fff

    class P p
    class I i
    class W w
```

**Golden rules applied:**

1. No segment trusts another by default.
2. The untrusted segment **never initiates** connections to the trusted one.
3. Traffic is evaluated by **interface + direction + service**.
4. **Default deny**, with the minimum necessary opened.

---

## 📡 DHCP & DNS

- **DHCP:** served by the **OpenWRT router** on each VLAN.
- **DNS:** handled internally by Pi-hole (see [server docs](./server.md)),
  with the OpenWRT router as the entry point.

---

## 📝 Notes

This document focuses on **design decisions and reasoning**, not on
exact configuration. IP ranges, VLAN IDs, SSIDs, and specific firewall
rules are intentionally omitted.
