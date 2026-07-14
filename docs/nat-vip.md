# NAT & Virtual IPs

## Source NAT with IP Pools

```text linenums="1"
config firewall ippool
    edit "POOL-Public"
        set type overload
        set startip 203.0.113.10
        set endip 203.0.113.14
    next
end

config firewall policy
    edit 10
        set nat enable
        set ippool enable
        set poolname "POOL-Public"
    next
end
```

- `type overload` — many-to-few PAT (default). Other types: `one-to-one`, `fixed-port-range`, `port-block-allocation` (CGNAT-style, logs port blocks instead of every session).

## Virtual IPs (DNAT)

```text linenums="1"
config firewall vip
    edit "VIP-WebSrv-443"
        set extip 203.0.113.20
        set extintf "wan1"
        set mappedip "10.10.10.5"
        set portforward enable
        set extport 443
        set mappedport 8443
        set protocol tcp
        set comment "Public HTTPS to internal 8443"
    next
    edit "VIP-Full-1to1"
        set extip 203.0.113.21
        set extintf "any"
        set mappedip "10.10.10.6"
        set comment "Static 1:1 NAT, all ports"
    next
end

config firewall policy
    edit 30
        set name "Inbound-Web"
        set srcintf "wan1"
        set dstintf "Z-INSIDE"
        set srcaddr "all"
        set dstaddr "VIP-WebSrv-443"
        set action accept
        set schedule "always"
        set service "HTTPS"
        set logtraffic all
    next
end
```

- The inbound policy's `dstaddr` is the **VIP object** (post-7.0 default `match-vip` behavior; on older configs a policy with `dstaddr all` and `set match-vip enable` also matches).
- Without `portforward`, the VIP maps **all** ports 1:1. With it, only `extport→mappedport`.
- `extintf any` lets the same mapping apply regardless of ingress interface.
- 1:1 VIPs also source-NAT outbound replies; for the internal host to *originate* traffic with the public IP, add `set nat-source-vip enable` or a matching IP pool.
- Hairpin/NAT-loopback: internal clients hitting the public IP need a policy `srcintf internal → dstintf internal` with the VIP as dstaddr.

| Command | Explanation |
|---|---|
| `diagnose firewall vip realserver list` | VIP/server-load-balance real server health. |
| `diagnose sys session list` (look for `hook=pre dir=org act=dnat`) | Confirm DNAT is applied to sessions. |
| `get firewall vip` | Summary of all VIPs. |

---