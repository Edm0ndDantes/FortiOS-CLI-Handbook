# Static Routing

```text
config router static
    edit 1
        set dst 0.0.0.0/0
        set gateway 203.0.113.1
        set device "wan1"
        set distance 10
        set priority 1
        set comment "Primary default via ISP-A"
    next
    edit 2
        set dst 0.0.0.0/0
        set gateway 198.51.100.1
        set device "wan2"
        set distance 20
        set comment "Backup default via ISP-B"
    next
    edit 3
        set dst 10.20.0.0/16
        set device "ipsec-hub"
        set comment "Route via IPsec tunnel interface (no gateway needed)"
    next
    edit 4
        set dst 0.0.0.0/0
        set device "wan1"
        set dynamic-gateway enable
        set comment "Default route learned from DHCP on wan1"
    next
    edit 5
        set blackhole enable
        set dst 10.99.99.0/24
        set distance 254
        set comment "Blackhole to prevent leaking via default"
    next
end
```

- `distance` — administrative distance. Lower wins. Two defaults with different distances = active/standby failover (backup installs only when primary route is removed, e.g. link down or link monitor failure).
- `priority` — tie-breaker among routes with **equal** distance; lower priority preferred, but both stay in the routing table (unlike distance). Used heavily with SD-WAN.
- Routes via tunnel interfaces (IPsec/GRE) need only `set device`; no gateway IP.
- `dynamic-gateway enable` — use the gateway learned by DHCP/PPPoE on that interface.
- `blackhole enable` — silently drop traffic to the prefix; standard practice for RFC1918 aggregates and IPsec-protected prefixes so traffic never escapes unencrypted when a tunnel drops.

**ECMP (equal-cost multipath):** two routes with identical dst/distance/priority load-balance. Algorithm:

```text
config system settings
    set v4-ecmp-mode source-ip-based   # or usage-based / weight-based / source-dest-ip-based
end
```

- `source-ip-based` (default) — per-source stickiness.
- `source-dest-ip-based` — 5-tuple-ish spread, better distribution.
- `weight-based` — proportional to `set weight` on each static route.

**Policy-based routing (PBR):**

```text
config router policy
    edit 1
        set input-device "vlan100-users"
        set src "10.100.0.0/255.255.255.0"
        set dst "0.0.0.0/0.0.0.0"
        set protocol 6
        set start-port 443
        set end-port 443
        set gateway 198.51.100.1
        set output-device "wan2"
        set comments "Steer user HTTPS via ISP-B"
    next
end
```

- PBR is checked **before** the routing table (except for local-out traffic). Matches on input interface, src/dst, protocol, ports, ToS.
- `action deny` variant can stop PBR processing and fall through to normal routing.
- The routing table must still contain a valid route out the chosen device, and a firewall policy must allow the flow.

| Command | Explanation |
|---|---|
| `get router info routing-table all` | Full RIB: connected, static, and dynamic routes with distance/metric. |
| `get router info routing-table details 8.8.8.8` | Which route (and ECMP set) matches a specific destination. |
| `get router info routing-table database` | All candidate routes, including inactive backups (shows why a route isn't installed). |
| `diagnose firewall proute list` | Installed policy routes (includes SD-WAN rules, which are PBR under the hood). |
| `diagnose ip rtcache list` | (Older builds) route cache entries. |

---