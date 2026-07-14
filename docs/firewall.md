# Firewall Objects & Policies

## Address & Service Objects

```text linenums="1"
config firewall address
    edit "NET-Users"
        set subnet 10.100.0.0 255.255.255.0
        set comment "User VLAN"
    next
    edit "HST-WebSrv"
        set subnet 10.10.10.5 255.255.255.255
    next
    edit "FQDN-Updates"
        set type fqdn
        set fqdn "updates.example.com"
    next
    edit "GEO-CY"
        set type geography
        set country "CY"
    next
end

config firewall addrgrp
    edit "GRP-Internal-Nets"
        set member "NET-Users" "HST-WebSrv"
    next
end

config firewall service custom
    edit "TCP-8443"
        set tcp-portrange 8443
    next
    edit "SVC-Syslog"
        set udp-portrange 514
    next
end
```

- Object types: `ipmask` (default), `iprange`, `fqdn` (resolved via DNS, TTL-tracked), `geography`, `mac`, `dynamic` (SDN connectors).
- FQDN objects depend on the FortiGate's DNS working and clients resolving to the **same** answers — a common gotcha behind split-horizon DNS.
- Groups keep policies readable and auditable.

## Firewall Policies

```text linenums="1"
config firewall policy
    edit 10
        set name "Users-to-Internet"
        set srcintf "Z-INSIDE"
        set dstintf "wan1"
        set srcaddr "NET-Users"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "HTTP" "HTTPS" "DNS"
        set nat enable
        set utm-status enable
        set ssl-ssh-profile "certificate-inspection"
        set av-profile "default"
        set webfilter-profile "default"
        set ips-sensor "default"
        set application-list "default"
        set logtraffic all
        set comments "Baseline user egress"
    next
    edit 20
        set name "Deny-Guest-to-Internal"
        set srcintf "vlan300-guest"
        set dstintf "Z-INSIDE"
        set srcaddr "all"
        set dstaddr "GRP-Internal-Nets"
        set action deny
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end
```

- Policies are evaluated **top-down, first match**; the implicit final policy denies everything (log it — see hardening).
- `nat enable` — source NAT to the egress interface IP (or use an IP pool: `set ippool enable`, `set poolname <name>`).
- `utm-status enable` + profiles — attach security profiles; `ssl-ssh-profile` is mandatory when any UTM profile is set (use `certificate-inspection` minimum, `deep-inspection` for full visibility).
- `logtraffic` — `all` (sessions + violations), `utm` (security events only), `disable`. Log denies on sensitive boundaries.
- Move policies: `move 20 before 10` at the `config firewall policy` level.

| Command | Explanation |
|---|---|
| `show firewall policy 10` | Show one policy's config. |
| `diagnose firewall iprope lookup <src-ip> <src-port> <dst-ip> <dst-port> <proto> <src-intf>` | Ask the box which policy a hypothetical packet would match — fastest "why is this blocked" tool. |
| `diagnose firewall iprope show 100004 <policy-id>` | Internal policy structure / hit counters. |
| `execute policy-lookup ...` (7.4+) | Friendlier policy lookup wrapper. |
| `diagnose sys session filter dst 8.8.8.8` + `diagnose sys session list` | Inspect live sessions matching a filter (shows matched `policy_id`). |

---