
# Dynamic Routing

## OSPF

```text linenums="1"
config router ospf
    set router-id 10.255.255.1
    config area
        edit 0.0.0.0
        next
        edit 0.0.0.10
            set type nssa
            set nssa-default-information-originate enable
        next
    end
    config network
        edit 1
            set prefix 10.10.10.0 255.255.255.0
            set area 0.0.0.0
        next
    end
    config ospf-interface
        edit "ospf-lan"
            set interface "internal1"
            set network-type broadcast
            set cost 10
            set hello-interval 10
            set dead-interval 40
            set authentication message-digest
            set md5-keys
            # (7.x) config md5-keys / edit 1 / set key-string <key>
        next
        edit "ospf-p2p-core"
            set interface "vlan400-core"
            set network-type point-to-point
        next
    end
    config redistribute "static"
        set status enable
        set metric 100
        set metric-type 2
    end
    set default-information-originate enable
end
```

- `router-id` — set explicitly (a loopback IP). Never rely on auto-selection; RID changes reset adjacencies.
- `config area` — define areas; `type` can be `regular`, `stub`, or `nssa`.
- `config network` — classic network statements binding interfaces to areas by prefix match.
- `config ospf-interface` — per-interface tuning: `network-type` (`broadcast`, `point-to-point`, `point-to-multipoint`, `non-broadcast`), timers, `cost`, and authentication. `message-digest` (MD5) or, on 7.4+, key-chains with SHA support.
- Use `point-to-point` on /30–/31 links — skips DR/BDR election and speeds convergence.
- `redistribute static/connected/bgp` — inject external routes; `metric-type 2` (default) keeps metric constant across the domain.
- `default-information-originate` — advertise 0.0.0.0/0 (add `always` to advertise even without a local default).

| Command | Explanation |
|---|---|
| `get router info ospf neighbor` | Adjacency table: neighbor RID, state (should be `Full`), interface. |
| `get router info ospf interface` | OSPF parameters per interface (area, type, DR/BDR, timers, cost). |
| `get router info ospf database brief` | LSDB summary per area. |
| `get router info routing-table ospf` | Only OSPF routes from the RIB. |
| `diagnose ip router ospf all enable` + `diagnose ip router ospf level info` | Enable OSPF daemon debug (view with `diagnose debug enable`). Disable when done. |

**Example — `get router info ospf neighbor`:**

```text linenums="1"
OSPF process 0, VRF 0:
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.255.2      1   Full/DR         00:00:36    10.10.10.2      internal1
10.255.255.3      1   Full/ -         00:00:33    10.40.0.2       vlan400-core
```

- `State Full` — full adjacency, LSDBs synchronized. `Full/DR` means the neighbor is the Designated Router on that broadcast segment; `Full/ -` on a point-to-point link (no DR election).
- Stuck in `ExStart/Exchange` → MTU mismatch. Stuck in `Init` → the neighbor doesn't see our hellos (auth mismatch, ACL, unicast/multicast filtering). No entry at all → area/subnet/timer mismatch or interface not in OSPF.

## BGP

```text linenums="1"
config router bgp
    set as 65010
    set router-id 10.255.255.1
    set ibgp-multipath enable
    set ebgp-multipath enable
    set graceful-restart enable
    config neighbor
        edit "203.0.113.1"
            set remote-as 65001
            set description "ISP-A transit"
            set prefix-list-in "PL-DEFAULT-ONLY"
            set route-map-out "RM-ADVERTISE-PUBLIC"
            set password <md5-secret>
            set ebgp-enforce-multihop disable
            set soft-reconfiguration enable
        next
        edit "10.255.255.2"
            set remote-as 65010
            set update-source "lo-mgmt"
            set next-hop-self enable
            set description "iBGP peer core-2"
        next
    end
    config network
        edit 1
            set prefix 192.0.2.0 255.255.255.0
        next
    end
    config redistribute "connected"
        set status enable
        set route-map "RM-CONN-TO-BGP"
    end
end
```

- `soft-reconfiguration enable` — stores received routes pre-policy so `clear ... soft` works without bouncing the session; costs memory but invaluable operationally.
- `update-source` + loopback peering for iBGP survives physical link failures (requires IGP reachability to the loopback).
- `next-hop-self` — rewrite next-hop toward iBGP peers so they don't need routes to external next-hops.
- `password` — TCP MD5 session protection; always use on eBGP over shared media.
- Always attach `prefix-list-in/out` or `route-map-in/out` to eBGP peers — never accept/advertise unfiltered.

**Route-maps and prefix-lists:**

```text linenums="1"
config router prefix-list
    edit "PL-DEFAULT-ONLY"
        config rule
            edit 1
                set prefix 0.0.0.0 0.0.0.0
                unset ge
                unset le
            next
        end
    next
end

config router route-map
    edit "RM-ADVERTISE-PUBLIC"
        config rule
            edit 1
                set match-ip-address "PL-PUBLIC"
                set set-community "65010:100"
            next
            edit 100
                set action deny
            next
        end
    next
end
```

- Prefix-list rules support `ge`/`le` for prefix length ranges.
- Route-map rules evaluate in order; an explicit final `deny` rule documents intent (implicit deny applies to route-maps used as filters).

| Command | Explanation |
|---|---|
| `get router info bgp summary` | Peer table: state, uptime, prefixes received. |
| `get router info bgp neighbors 203.0.113.1 advertised-routes` | What we advertise to a peer (post-policy). |
| `get router info bgp neighbors 203.0.113.1 routes` | What we accepted from a peer. |
| `get router info bgp network 8.8.8.0/24` | BGP table detail for a prefix (paths, attributes, bestpath reason). |
| `execute router clear bgp ip 203.0.113.1 soft in` | Re-apply inbound policy without tearing the session down. |
| `diagnose ip router bgp all enable` | BGP daemon debug (pair with `diagnose debug enable`). |

**Example — `get router info bgp summary`:**

```text linenums="1"
BGP router identifier 10.255.255.1, local AS number 65010
Neighbor        V   AS     MsgRcvd MsgSent  TblVer InQ OutQ Up/Down    State/PfxRcd
203.0.113.1     4   65001   152344  151201       9   0    0  5d02h11m         187
10.255.255.2    4   65010    88123   88101       9   0    0  5d02h09m          42
```

- A **number** in `State/PfxRcd` = session Established, showing prefixes received.
- `Active`/`Connect` — TCP can't establish (routing/ACL/policy to peer, wrong peer IP). `Idle` — misconfig or admin down. Flapping with `OpenConfirm` → auth/AS mismatch or hold-timer expiry (check MTU/CPU).

---