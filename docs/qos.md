# Traffic Shaping & QoS

## Shared & Per-IP Shapers

```text
config firewall shaper traffic-shaper
    edit "SH-Guest-50M"
        set guaranteed-bandwidth 0
        set maximum-bandwidth 51200
        set bandwidth-unit kbps
        set per-policy enable
        set priority low
    next
    edit "SH-Voice"
        set guaranteed-bandwidth 10240
        set maximum-bandwidth 20480
        set priority high
        set diffserv enable
        set diffservcode 101110        # EF
    next
end

config firewall shaper per-ip-shaper
    edit "PIP-Guest-5M"
        set max-bandwidth 5120
        set bandwidth-unit kbps
        set max-concurrent-session 500
    next
end
```

- `guaranteed-bandwidth` — reserved under congestion; `maximum-bandwidth` — hard cap (0 = unlimited guarantee/cap depending on field).
- `per-policy` — each policy using the shaper gets its own instance; disabled = one bucket shared by every policy referencing it.
- `priority` — high/medium/low scheduling among shaped traffic once guarantees are met.
- `diffservcode` — rewrite DSCP on egress (EF = `101110`, AF41 = `100010`...), aligning upstream QoS.
- Per-IP shapers cap **each source IP** individually — ideal for guest networks.

## Traffic Shaping Policies (v6.4+ model)

```text
config firewall shaping-policy
    edit 1
        set name "Shape-Voice"
        set service "SIP" "RTP"
        set dstintf "wan1"
        set srcaddr "all"
        set dstaddr "all"
        set traffic-shaper "SH-Voice"
        set traffic-shaper-reverse "SH-Voice"
    next
    edit 2
        set name "Cap-Guest"
        set srcintf "vlan300-guest"
        set dstintf "wan1"
        set srcaddr "all"
        set dstaddr "all"
        set per-ip-shaper "PIP-Guest-5M"
    next
end
```

- Shaping policies match traffic (interfaces, addresses, services, applications, URL categories) **after** the firewall policy allows it, and attach shapers. This decouples shaping from security policy.
- `traffic-shaper` = forward (client→server) direction; `traffic-shaper-reverse` = reply direction — set both for symmetric control.

## Interface Egress Shaping & Shaping Profiles (CBQ-style)

```text
config firewall shaping-profile
    edit "PROF-WAN"
        set type policing            # or "queuing" on supported NP7/SoC ports
        config shaping-entries
            edit 1
                set class-id 2
                set guaranteed-bandwidth-percentage 30
                set maximum-bandwidth-percentage 100
                set priority high
            next
            edit 2
                set class-id 3
                set guaranteed-bandwidth-percentage 10
                set maximum-bandwidth-percentage 50
                set priority low
            next
        end
        set default-class-id 3
    next
end

config system interface
    edit "wan1"
        set outbandwidth 100000
        set egress-shaping-profile "PROF-WAN"
        set inbandwidth 500000
    next
end
```

- `outbandwidth` on the interface defines the real link rate (kbps) — the profile percentages divide **this**. Without it the profile does nothing meaningful.
- Traffic is mapped to `class-id`s by shaping policies (`set class-id 2` in a shaping-policy rule); the profile then guarantees/caps each class.
- `inbandwidth` polices ingress (crude, drops after arrival — shaping is only truly effective egress).

| Command | Explanation |
|---|---|
| `diagnose firewall shaper traffic-shaper list` | All shapers with live rate, drops, and current bandwidth. |
| `diagnose firewall shaper per-ip-shaper list` | Per-IP shaper state and counters. |
| `diagnose netlink interface list wan1` | Look for `qdisc` and drop counters on the shaped interface. |
| `diagnose sys traffic-shaping ...` (7.4+) | Newer consolidated shaping diagnostics. |

**Example — `diagnose firewall shaper traffic-shaper list` (one entry):**

```text
name SH-Voice
maximum-bandwidth 2560 KB/sec
guaranteed-bandwidth 1280 KB/sec
current-bandwidth 96 KB/sec
priority 1
overhead 0
tos ff
packets dropped 0
bytes dropped 0
```

- Note the units: the diagnostic shows **KB/sec** (kilobytes) while config is kbps (kilobits) — divide config by 8.
- `current-bandwidth` vs `maximum` tells you utilization; non-zero `packets dropped` means the cap is being hit.
