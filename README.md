# 🌐 Multi-VLAN Enterprise Network — Cisco Packet Tracer

A fully functional enterprise network simulation built in Cisco Packet Tracer, featuring VLAN segmentation, Router-on-a-Stick inter-VLAN routing, a dedicated DHCP server, OSPF dynamic routing, and Extended ACLs with security policies.

Built as part of my preparation for a **Cisco Customer Experience (CX) Engineer Internship** interview.

---

## 📋 Network Design

| VLAN | Name       | Subnet            | Gateway       | Devices                        |
|------|------------|-------------------|---------------|--------------------------------|
| 10   | Management | 192.168.10.0/24   | 192.168.10.1  | Management PC, DHCP Server     |
| 20   | Staff      | 192.168.20.0/24   | 192.168.20.1  | Staff PC                       |
| 30   | Guest      | 192.168.30.0/24   | 192.168.30.1  | Guest PC                       |

**Devices:**
- Cisco 2911 Router
- Cisco 2960 Switch
- 3x PCs (one per VLAN)
- Server-PT (dedicated DHCP server, static IP 192.168.10.2)

---

## 🛠️ Features Implemented

### ✅ VLAN Segmentation
Three VLANs created and named on the switch. Each VLAN represents a different trust zone — Management (IT admins), Staff (employees), and Guest (visitors).

### ✅ Router-on-a-Stick
One physical router interface (`GigabitEthernet 0/0`) with three subinterfaces, each handling a separate VLAN using 802.1Q encapsulation.

### ✅ Dedicated DHCP Server (Enterprise Architecture)
A centralized DHCP server on the Management VLAN serves all three VLANs. The router uses `ip helper-address` to relay DHCP broadcasts from VLAN 20 and 30 to the server on VLAN 10. VLAN 10 devices reach the server directly without relay.

### ✅ Extended ACLs with DHCP Exception
`GUEST_RESTRICT` ACL applied inbound on the Guest subinterface:
- ✅ Permits DHCP traffic (UDP ports 67/68) before deny rules
- ❌ Blocks Guest → Management (192.168.10.0/24)
- ❌ Blocks Guest → Staff (192.168.20.0/24)
- ✅ Permits Guest → internet (any other destination)

### ✅ OSPF Dynamic Routing
OSPF process 1 configured with all three networks in Area 0. Ready to scale — adding a second router will automatically exchange routes without static route configuration.

---

## 📁 Repository Structure

```
multi-vlan-enterprise-home-lab/
├── README.md
├── topology.png                        ← Full topology screenshot
├── Homelab Network.pkt           ← Packet Tracer file
├── configs/
│   ├── router-config.txt              ← Full router running-config
│   └── switch-config.txt             ← Full switch running-config
└── screenshots/
    ├── show-vlan-brief.png
    ├── show-interfaces-trunk.png
    ├── show-ip-interface-brief.png
    ├── show-ip-route.png
    ├── show-access-lists.png
    ├── guest-failed-ping.png
    └── staff-successful-ping.png
```

---

## ⚙️ Key Configurations

### Switch — VLAN Creation
```
vlan 10
 name Management
vlan 20
 name Staff
vlan 30
 name Guest
```

### Switch — Trunk Port
```
interface GigabitEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

### Router — Subinterfaces (Router-on-a-Stick)
```
interface GigabitEthernet 0/0
 no shutdown

interface GigabitEthernet 0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface GigabitEthernet 0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.10.2

interface GigabitEthernet 0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 192.168.10.2
```

### Router — OSPF
```
router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
```

### Router — ACL
```
ip access-list extended GUEST_RESTRICT
 10 permit udp any any eq bootps
 20 permit udp any any eq bootpc
 30 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 40 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 50 permit ip 192.168.30.0 0.0.0.255 any

interface GigabitEthernet 0/0.30
 ip access-group GUEST_RESTRICT in
```

---

## 🔍 Verification Results

| Test | From | To | Result |
|---|---|---|---|
| Gateway reachability | Management PC | 192.168.10.1 | ✅ Success |
| Inter-VLAN routing | Management PC | 192.168.20.11 (Staff) | ✅ Success |
| Inter-VLAN routing | Staff PC | 192.168.10.11 (Management) | ✅ Success |
| ACL blocking | Guest PC | 192.168.10.11 (Management) | ❌ Blocked ✅ |
| ACL blocking | Guest PC | 192.168.20.11 (Staff) | ❌ Blocked ✅ |
| Gateway reachability | Guest PC | 192.168.30.1 | ✅ Success |
| DHCP | All PCs | DHCP server | ✅ All received IPs |

**ACL hit counts (show access-lists):**
```
30 deny ip 192.168.30.0 → 192.168.10.0  (15 matches)
40 deny ip 192.168.30.0 → 192.168.20.0  (4 matches)
```

---

## 🧠 Key Lessons Learned

1. **APIPA (169.254.x.x)** — Self-assigned fallback IP when DHCP fails. First troubleshooting step is to ping the default gateway.

2. **ip helper-address placement** — Only needed on subinterfaces whose VLAN is on a different subnet than the DHCP server. Never needed on the same subnet.

3. **ACL blocking DHCP relay** — Extended ACLs can silently block DHCP relay traffic. Fix: explicitly permit UDP ports 67 (bootps) and 68 (bootpc) before deny rules.

4. **Inbound ACL placement** — Applying ACLs inbound on the source interface is most efficient — traffic is filtered before the router wastes resources routing it.

5. **Router-on-a-Stick** — The physical interface has no IP. Subinterfaces carry the gateway IPs. The trunk link carries all VLAN traffic tagged with 802.1Q.

---

## 🎯 Skills Demonstrated

- VLAN design and segmentation
- 802.1Q trunking
- Router-on-a-Stick inter-VLAN routing
- Enterprise DHCP architecture with relay (ip helper-address)
- Extended ACL design with security exceptions
- OSPF dynamic routing configuration
- Systematic network troubleshooting methodology
- Network documentation

---

## 👤 Author

**Bahaa Ahmed Yahya**
- 📍 Cairo, Egypt
- 🎓 Computer & Control Engineering — El-Shorouk Academy (2026)
- 🏆 Cisco CCNA Certified | AWS Cloud Practitioner
- 🔗 [LinkedIn](https://www.linkedin.com/in/bahaa-ahmed-/)
- 💻 [GitHub](https://github.com/BahaaAhmedOfficial)
