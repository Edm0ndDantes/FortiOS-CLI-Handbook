# GRE Tunnels

```text
config system gre-tunnel
    edit "GRE-DC2"
        set interface "wan1"
        set local-gw 203.0.113.5
        set remote-gw 192.0.2.77
        set keepalive-interval 10
        set keepalive-failtimes 3
    next
end

config system interface
    edit "GRE-DC2"
        set ip 172.16.99.1 255.255.255.255
        set remote-ip 172.16.99.2 255.255.255.252
        set allowaccess ping
    next
end

config router static
    edit 30
        set dst 10.30.0.0/16
        set device "GRE-DC2"
    next
end

config firewall policy
    edit 50
        set name "LAN-to-GRE-DC2"
        set srcintf "Z-INSIDE"
        set dstintf "GRE-DC2"
        set srcaddr "GRP-Internal-Nets"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

- GRE gives protocol flexibility (multicast, non-IP payloads for routing protocols) but **no encryption** — pair with IPsec (GRE-over-IPsec) for confidentiality.
- `keepalive-interval` / `keepalive-failtimes` — GRE keepalives so the interface goes down when the far end dies (otherwise GRE is stateless and stays "up").
- The tunnel interface config (IP/remote-ip, routes, policies) mirrors the IPsec pattern.

**GRE over IPsec:** build the IPsec phase1/phase2 with selectors matching the two GRE endpoints (host-to-host, `set protocol 47` on phase2 optionally), set the GRE `local-gw/remote-gw` to those same endpoints, and route the GRE endpoints via the IPsec tunnel. Multicast routing (PIM/OSPF) then runs over GRE while IPsec encrypts everything.

| Command | Explanation |
|---|---|
| `diagnose sys gre list` | GRE tunnel kernel state and counters. |
| `diagnose sniffer packet any 'proto 47' 4` | Confirm GRE packets on the wire. |

---