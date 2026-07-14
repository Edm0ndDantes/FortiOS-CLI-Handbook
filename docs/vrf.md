# VRFs

```text
config system global
    set vrf-mode enable            # not required on newer builds; VRFs 0-251 available by default in 7.x
end

config system interface
    edit "vlan100-users"
        set vrf 10
    next
    edit "vlan300-guest"
        set vrf 20
    next
    edit "lo-vrf10"
        set type loopback
        set vrf 10
        set ip 10.255.10.1 255.255.255.255
    next
end

config router static
    edit 10
        set dst 0.0.0.0/0
        set gateway 10.100.0.254
        set device "vlan100-users"
        set vrf 10
    next
end
```

- `set vrf <0-251>` on an interface places it (and its connected routes) into that VRF's routing table. VRF 0 is the default/global table.
- Static routes carry a `set vrf` attribute; each VRF keeps a fully separate RIB/FIB.
- BGP supports per-VRF configuration and inter-VRF route leaking (`config router bgp → config vrf / config vrf-leak` depending on version) for controlled communication between VRFs.
- Traffic between interfaces in **different** VRFs will not route unless leaked — VRFs give you true L3 isolation inside one VDOM (the classic alternative to multi-VDOM designs).
- Management traffic can be pinned to a VRF with `set vrf-select` / management VRF options on 7.4+.

| Command | Explanation |
|---|---|
| `get router info routing-table all vrf 10` | Show the routing table for VRF 10 only. |
| `diagnose ip route list table 10` | Kernel FIB view of a specific VRF table. |
| `execute ping-options ...` then `execute ping <ip>` | Pings originate in VRF of the selected source interface (`execute ping-options interface <if>`). |

---