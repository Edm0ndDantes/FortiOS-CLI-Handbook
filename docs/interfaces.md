# Interfaces

## Physical Interface Basics

```text
config system interface
    edit "internal1"
        set mode static
        set ip 10.10.10.1 255.255.255.0
        set allowaccess ping
        set alias "LAN-Users"
        set role lan
        set description "User access segment"
        set device-identification enable
    next
    edit "wan1"
        set mode dhcp                  # or static / pppoe
        set role wan
        set estimated-upstream-bandwidth 100000
        set estimated-downstream-bandwidth 500000
    next
end
```

- `mode` — `static`, `dhcp`, or `pppoe` (PPPoE adds `username`/`password`).
- `alias` — friendly name shown in logs and GUI; keeps configs readable.
- `role` — `lan`, `wan`, `dmz`, `undefined`; used by GUI logic and SD-WAN, cosmetic on the CLI but keep it accurate.
- `device-identification` — enables device detection on the segment (useful for logging/ZTNA-style visibility).
- `estimated-*-bandwidth` — in kbps; informs SD-WAN and shaping calculations.

## VLAN Interfaces (802.1Q)

```text
config system interface
    edit "vlan100-users"
        set vdom "root"
        set interface "internal1"
        set vlanid 100
        set ip 10.100.0.1 255.255.255.0
        set allowaccess ping
        set alias "Users-VLAN100"
    next
    edit "vlan200-voice"
        set vdom "root"
        set interface "internal1"
        set vlanid 200
        set ip 10.200.0.1 255.255.255.0
        set vlan-protocol 8021q
    next
end
```

- A VLAN interface is a subinterface bound to a parent (`interface "internal1"`). The parent carries the tagged trunk.
- `vlanid` — 1–4094; must match the switch trunk allowed VLANs.
- `vlan-protocol` — `8021q` (default) or `8021ad` (QinQ outer tag).
- Untagged traffic on the trunk is handled by the parent interface itself.
- Each VLAN interface is a full L3 interface: it can hold policies, DHCP servers, routing, etc.

## Loopback Interfaces

```text
config system interface
    edit "lo-mgmt"
        set vdom "root"
        set type loopback
        set ip 10.255.255.1 255.255.255.255
        set allowaccess ping https ssh
    next
end
```

- `type loopback` — always-up virtual interface. Ideal for: stable management IP, BGP/OSPF router-ID, IPsec/GRE tunnel source, RADIUS/TACACS `source-ip`, SNMP source.
- Reachability requires either an IGP advertising it or static routes on peers; firewall policies **to** the loopback interface are needed for management access through the box.

## Link Aggregation (LACP / 802.3ad)

```text
config system interface
    edit "lag1"
        set vdom "root"
        set type aggregate
        set member "port3" "port4"
        set lacp-mode active
        set lacp-speed fast
        set algorithm L4
        set min-links 1
        set min-links-down operational
        set ip 10.30.0.1 255.255.255.0
        set allowaccess ping
    next
end
```

- `type aggregate` — creates an 802.3ad LAG. Member ports must have no existing config/references and identical speed/duplex.
- `lacp-mode` — `active` (sends LACPDUs), `passive` (responds only), `static` (no LACP; plain bundling — avoid unless peer can't do LACP).
- `lacp-speed` — `fast` = LACPDU every 1 s (3 s failure detect), `slow` = every 30 s.
- `algorithm` — hash for egress member selection: `L2` (MAC), `L3` (IP), `L4` (IP+port; best distribution for many flows).
- `min-links` — minimum active members before the LAG is declared down (`min-links-down operational` marks it operationally down so routing fails over).
- VLAN subinterfaces can ride on top of the LAG exactly as on a physical port.

## Zones

```text
config system zone
    edit "Z-INSIDE"
        set interface "vlan100-users" "vlan200-voice" "lag1"
        set intrazone deny
    next
end
```

- Zones group interfaces so firewall policies reference one object instead of many interfaces.
- `intrazone deny` — traffic between member interfaces requires explicit policy (recommended). `allow` permits it implicitly.
- Once an interface is in a zone, policies must reference the **zone**, not the interface.

## Software Switch (use sparingly)

```text
config system switch-interface
    edit "sw-lan"
        set vdom "root"
        set member "port5" "port6"
    next
end
```

- Bridges ports into one L2/L3 switch interface. All traffic between members is processed in software — CPU cost. Prefer hardware switch (`virtual-switch` on supported models) or a real switch.

| Command | Explanation |
|---|---|
| `get system interface physical` | Physical link state, speed/duplex, MAC, IP of every port. |
| `diagnose netlink interface list` | Kernel-level interface flags, indexes, and errors counters. |
| `diagnose netlink aggregate name lag1` | LACP state per member: actor/partner state, selected/collecting/distributing bits. |
| `diagnose hardware deviceinfo nic port1` | Driver-level NIC counters (CRC errors, drops) — first stop for suspected L1 issues. |
| `execute interface dhcpclient-renew wan1` | Renew the DHCP lease on a DHCP-mode interface. |

---