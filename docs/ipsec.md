# IPsec VPN

## Route-Based Site-to-Site (IKEv2, static peers)

```text linenums="1"
config vpn ipsec phase1-interface
    edit "S2S-BRANCH1"
        set interface "wan1"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal aes256-sha256 aes256gcm-prfsha384
        set dhgrp 20 19
        set nattraversal enable
        set dpd on-idle
        set dpd-retryinterval 5
        set remote-gw 198.51.100.50
        set psksecret <pre-shared-key>
        set keylife 86400
    next
end

config vpn ipsec phase2-interface
    edit "S2S-BRANCH1-P2"
        set phase1name "S2S-BRANCH1"
        set proposal aes256-sha256 aes256gcm
        set dhgrp 20 19
        set pfs enable
        set auto-negotiate enable
        set keylifeseconds 3600
        set src-subnet 10.0.0.0 255.0.0.0
        set dst-subnet 10.20.0.0 255.255.0.0
    next
end

config system interface
    edit "S2S-BRANCH1"
        set ip 10.254.1.1 255.255.255.255
        set remote-ip 10.254.1.2 255.255.255.252
        set allowaccess ping
    next
end

config router static
    edit 20
        set dst 10.20.0.0/16
        set device "S2S-BRANCH1"
    next
    edit 21
        set blackhole enable
        set dst 10.20.0.0/16
        set distance 254
    next
end

config firewall policy
    edit 40
        set name "LAN-to-Branch1"
        set srcintf "Z-INSIDE"
        set dstintf "S2S-BRANCH1"
        set srcaddr "GRP-Internal-Nets"
        set dstaddr "NET-Branch1"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 41
        set name "Branch1-to-LAN"
        set srcintf "S2S-BRANCH1"
        set dstintf "Z-INSIDE"
        set srcaddr "NET-Branch1"
        set dstaddr "GRP-Internal-Nets"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end
```

- `phase1-interface` (route-based) creates a virtual tunnel interface named after the phase1 — always prefer this over legacy policy-based VPN.
- `ike-version 2` — use IKEv2 unless the peer can't; simpler negotiation, better DPD/rekey behavior.
- `proposal` / `dhgrp` — offer only strong suites: AES-256(-GCM), SHA-256+, DH groups 19/20/21 (ECP) or 14 minimum. Avoid 3DES/SHA1/DH1-2-5.
- `net-device disable` — single tunnel interface shared by all dialup peers (dynamic routing friendly); `enable` creates one per peer.
- `dpd on-idle` — Dead Peer Detection probes only when idle; fast tunnel-down detection with `dpd-retryinterval`.
- Phase 2 selectors (`src-subnet`/`dst-subnet`) must mirror each other across the two peers; `0.0.0.0/0 ↔ 0.0.0.0/0` is common for route-based tunnels and lets routing decide what enters the tunnel.
- `auto-negotiate` — bring the SA up proactively and rekey before expiry (avoids first-packet loss).
- The tunnel interface `ip`/`remote-ip` gives you a routable /30-style link for pings, dynamic routing (OSPF/BGP over the tunnel), and monitoring.
- Blackhole route (distance 254) prevents traffic escaping via the default route if the tunnel drops.
- Two policies (one per direction) are always required — IPsec is not exempt from the policy engine.

## IPsec with BGP over the Tunnel (hub-and-spoke friendly)

```text linenums="1"
config router bgp
    config neighbor
        edit "10.254.1.2"
            set remote-as 65020
            set description "Branch1 over IPsec"
            set update-source "S2S-BRANCH1"     # tunnel interface
        next
    end
end
```

- Peer using the tunnel interface addresses; routes learned via BGP replace static tunnel routes — this scales to many spokes and enables ADVPN later.

## Dialup / Dynamic-Peer Phase 1

```text linenums="1"
config vpn ipsec phase1-interface
    edit "DIALUP-SPOKES"
        set type dynamic
        set interface "wan1"
        set ike-version 2
        set peertype one
        set peerid "spoke-group-1"
        set add-route enable
        set proposal aes256-sha256
        set psksecret <key>
    next
end
```

- `type dynamic` — accepts connections from any source IP (spokes behind DHCP/NAT).
- `peerid`/`peertype` — restrict which IKE IDs may connect.
- `add-route enable` — inject reverse routes from the spoke's phase2 selectors automatically.

| Command | Explanation |
|---|---|
| `get vpn ipsec tunnel summary` | One line per tunnel: peer, selectors, up/down. |
| `diagnose vpn ike gateway list name S2S-BRANCH1` | IKE SA detail: state, version, negotiated algorithms, NAT-T. |
| `diagnose vpn tunnel list name S2S-BRANCH1` | IPsec SA detail: SPIs, encryption, packet/byte counters, npu-flag (offload state). |
| `diagnose vpn ike log-filter dst-addr4 198.51.100.50` (7.4+: `diagnose vpn ike log filter rem-addr4 ...`) | Filter IKE debug to one peer. |
| `diagnose debug application ike -1` + `diagnose debug enable` | Full IKE negotiation debug — the definitive tool for phase1/phase2 failures. Disable with `diagnose debug disable` and `diagnose debug reset`. |
| `execute vpn ipsec tunnel down S2S-BRANCH1` / `... up` | Bounce a tunnel manually. |

**Example — `diagnose vpn tunnel list name S2S-BRANCH1` (truncated):**

```text linenums="1"
name=S2S-BRANCH1 ver=2 serial=3 203.0.113.5:0->198.51.100.50:0 tun_id=10.254.1.2
bound_if=5 lgwy=static/1 tun=intf/0 mode=auto/1 encap=none/528 options[0210]=create_dev frag-rfc
proxyid_num=1 child_num=0 refcnt=14 ilast=2 olast=2 ad=/0
stat: rxp=182734 txp=190211 rxb=98123441 txb=101233812
dpd: mode=on-idle on=1 idle=5000ms retry=3 count=0 seqno=4211
proxyid=S2S-BRANCH1-P2 proto=0 sa=1 ref=2 serial=1 auto-negotiate
  src: 0:10.0.0.0/255.0.0.0:0
  dst: 0:10.20.0.0/255.255.0.0:0
  SA: ref=3 options=18227 type=00 soft=0 mtu=1438 expire=2314/0B replaywin=2048
       seqno=2f45 esn=0 replaywin_lastseq=00002e11 qat=0 rekey=0 hash_search_len=1
  dec: spi=8a1b2c3d esp=aes key=32 ...
  enc: spi=4d3c2b1a esp=aes key=32 ...
```

- `sa=1` — phase 2 SA established (0 = down, 2 = negotiating/duplicate).
- `rxp/txp` counters incrementing in **both** directions = traffic flowing. `txp` rising with `rxp` static = we send, peer (or its policies/routing) doesn't return traffic.
- `mtu=1438` — effective tunnel MTU; relevant for fragmentation/MSS issues.
- `dpd ... count=0` — DPD healthy; a rising count means the peer stopped answering.
- `expire` — seconds until rekey.

---