
# Dynamic Routing

### OSPF

**Theory recap in one paragraph:** OSPF is a link-state IGP (RFC 2328). Every router floods Link-State Advertisements (LSAs) describing its links; all routers in an area build an identical Link-State Database (LSDB) and independently run Dijkstra's SPF algorithm to compute shortest paths, using **cost** as the metric. Routers discover each other with multicast **Hello** packets (224.0.0.5), progress through an adjacency state machine (Down → Init → 2-Way → ExStart → Exchange → Loading → **Full**), and on multi-access segments elect a **Designated Router (DR)** and **Backup DR (BDR)** to reduce adjacency count from O(n²) to O(n). Areas bound LSA flooding; area 0.0.0.0 is the **backbone** that all other areas must attach to. Routers injecting external routes (e.g. redistributed BGP/static) are **ASBRs**; routers joining areas are **ABRs**.

#### OSPF.1 Process-Level Configuration

```text
config router ospf
    set router-id 10.255.255.1
    set default-information-originate enable    # Advertise 0.0.0.0/0 as a Type-5 LSA
    set distance 110                            # Administrative distance of OSPF routes
    set spf-timers 5 10                         # SPF schedule delay / hold between runs (s)
    set rfc1583-compatible disable              # External path preference per RFC 2328 (modern)
    set restart-mode graceful-restart           # Hitless restart / HA failover support
    set restart-period 120
end
```

- `router-id` — the 32-bit identifier under which this router appears in **every LSA it originates** and in the LSDB. OSPF theory: the RID must be unique in the domain; it also breaks ties in DR/BDR election. FortiOS will auto-pick an interface IP if unset, but always set it explicitly (conventionally a loopback IP) — an RID change forces all adjacencies and the router's LSAs to be rebuilt.
- `default-information-originate` — makes this router originate `0.0.0.0/0` as a **Type-5 AS-external LSA**, turning it into an ASBR. By default it only advertises the default if one exists in its own routing table (e.g. learned via BGP from the ISP); append `always` to originate unconditionally. This is exactly the pattern in Fortinet's basic example, where the BGP-facing router advertises the default into the OSPF domain.
- `distance` — administrative distance (110 by default) decides how OSPF routes compete against other **protocols** in the RIB. This is FortiOS route-selection, not OSPF theory: inside OSPF, path selection is purely by cost and LSA type (intra-area O > inter-area IA > E1 > E2).
- `spf-timers <delay> <hold>` — SPF throttling. Link-state theory: every LSDB change requires a full (or incremental) Dijkstra re-run; the delay dampens churn during flapping at the price of convergence speed.
- `rfc1583-compatible` — changes tie-breaking for AS-external paths. Must match on **all** routers in the domain or routing loops for external routes are possible; `disable` (RFC 2328 behavior) is the modern choice.
- `restart-mode graceful-restart` — implements OSPF Graceful Restart (RFC 3623): during a restart/HA failover the router's neighbors keep forwarding on its behalf ("helper mode") instead of tearing down adjacencies and reflooding, avoiding a domain-wide SPF event.

#### OSPF.2 Areas

```text
config router ospf
    config area
        edit 0.0.0.0                            # Backbone — mandatory transit for all areas
        next
        edit 0.0.0.51
            set type stub                       # or: nssa / regular
            set default-cost 10                 # Cost of the injected ::/0 summary into the stub
            set authentication message-digest   # Area-wide auth default (overridable per interface)
        next
    end
end
```

- `edit <area-id>` — areas are LSA flooding boundaries. Type-1 (Router) and Type-2 (Network) LSAs never leave their area; ABRs summarize reachability between areas as **Type-3 Summary LSAs**. Area `0.0.0.0` is the backbone: OSPF's loop-prevention model requires all inter-area traffic to transit it.
- `type regular` — normal area: carries Type-3 summaries and Type-5 externals.
- `type stub` — blocks **Type-5 external LSAs**; the ABR injects a default Type-3 route instead (its cost = `default-cost`). Use where routers don't need full external routing knowledge — shrinks LSDB and SPF load.
- `type nssa` — "not-so-stubby": like a stub, but still allows local redistribution. Externals injected inside the area travel as **Type-7 LSAs**, which the ABR translates to Type-5 at the area edge. `set nssa-default-information-originate enable` injects a default into the NSSA.
- `authentication` — area-level default for OSPFv2 packet authentication (`none`/`text`/`message-digest`); per-interface settings override it. Authentication protects the Hello/LSU exchange from rogue adjacency injection.

#### OSPF.3 Enabling OSPF: Network Statements

```text
config router ospf
    config network
        edit 1
            set prefix 10.11.0.0 255.255.0.0
            set area 0.0.0.0
        next
        edit 2
            set prefix 192.168.102.0 255.255.255.0
            set area 0.0.0.0
        next
    end
end
```

- A `network` entry does two things at once, exactly as in classic OSPF theory: any interface whose IP falls inside `prefix` (1) starts sending/listening for Hellos, i.e. runs OSPF, and (2) has its connected subnet described in this router's **Type-1 Router LSA** in the given `area`.
- A broad statement like `10.11.0.0/16` enrolls every matching interface at once (Fortinet's example uses this to cover all inter-router links); use exact /24s or interface-specific statements when you want tighter control over where adjacencies can form.
- To advertise a subnet **without** forming adjacencies on it (e.g. a user VLAN), keep it in a network statement but make the interface passive:

```text
config router ospf
    set passive-interface "VLAN100-Users"
end
```

- Passive interfaces suppress Hellos (no neighbors possible) but the prefix stays in the Router LSA — the standard way to advertise stub networks while eliminating the attack/misconfig surface of unnecessary adjacencies.

#### OSPF.4 Per-Interface Parameters (`ospf-interface`)

```text
config router ospf
    config ospf-interface
        edit "OSPF-Core-P2P"
            set interface "Agg1"
            set network-type point-to-point     # broadcast | point-to-point | p2mp | non-broadcast
            set cost 10                         # SPF metric of this link (0 = auto)
            set hello-interval 10               # Must match neighbor
            set dead-interval 40                # Must match neighbor (typ. 4x hello)
            set priority 255                    # DR election weight (0 = never DR)
            set authentication message-digest
            set md5-keys                        # config md5-keys → edit <id> → set key-string
            set mtu-ignore disable
        next
    end
end
```

- `network-type` — tells OSPF what the L2 segment looks like, which drives its behavior:
  - `broadcast` (default on Ethernet) — Hellos are multicast, a **DR/BDR is elected**, and all routers form full adjacencies only with DR/BDR (everyone else stays in 2-Way). The DR originates the **Type-2 Network LSA** representing the segment.
  - `point-to-point` — **no DR/BDR election**, adjacency forms directly. Use on any Ethernet link with exactly two routers (/30, /31): faster convergence and one less failure mode. This is why the neighbor state shows `Full/ -` on P2P links but `Full/DR` or `Full/Backup` on broadcast segments.
  - `point-to-multipoint` / `non-broadcast` — for NBMA topologies (rare today); neighbors may need static definition under `config neighbor`.
- `priority` — DR election input on broadcast segments: highest priority wins, ties broken by highest RID; `0` makes the router ineligible. Fortinet's example sets 255 on the intended DR and 250 on the intended BDR. Note OSPF's election is **non-preemptive** — a new higher-priority router won't displace a sitting DR until the DR fails, so priorities only guarantee outcomes if set before the segment converges.
- `cost` — OSPF's metric. SPF computes each route's total cost as the **sum of outgoing interface costs** along the path. FortiOS: `0` = auto-derive from bandwidth (reference-bandwidth ÷ interface bandwidth); set explicit values on links you steer traffic with. Equal total costs to the same prefix → ECMP (as seen when a destination shows two next-hops with identical `[110/2]`).
- `hello-interval` / `dead-interval` — the adjacency keepalive machinery. Both values are carried **inside the Hello packet** and *must match* between neighbors, or Hellos are rejected and the pair never leaves Init/Down — the single most common OSPF misconfig. Dead-interval (typically 4× hello) is how long silence is tolerated before the neighbor is declared down and LSAs are reflooded.
- `authentication` + `md5-keys` — per-interface auth (overrides the area default). `message-digest` puts an MD5 hash in every OSPF packet: neighbors with wrong/missing keys stay invisible (stuck Down/Init). Newer builds also support keychains/HMAC-SHA via `set keychain`.
- `mtu-ignore` — during **ExStart**, Database Description packets carry the interface MTU; a mismatch stalls the adjacency in ExStart/Exchange by design (to prevent un-fragmentable LSU loss). `enable` skips the check — a workaround, but fixing the MTU is the correct repair.

#### OSPF.5 Redistribution (ASBR Role) & Summarization

```text
config router ospf
    config redistribute "bgp"
        set status enable
        set metric 10
        set metric-type 2                       # E2 (fixed metric) | 1 = E1 (metric grows)
        set routemap "RM-BGP-TO-OSPF"           # Filter/tag what enters OSPF
    end
    config summary-address
        edit 1
            set prefix 172.16.0.0 255.240.0.0   # Summarize externals at the ASBR
        next
    end
end
```

- `config redistribute` (`bgp`, `static`, `connected`, `rip`, `isis`) — makes the router an **ASBR**: each imported route is originated as a **Type-5 AS-external LSA** flooded through all regular areas.
- `metric-type` — pure OSPF theory: **E2** (default) carries a fixed external metric that never changes as it propagates — every router sees the same cost, so path choice toward the prefix follows cost-to-ASBR only as a tie-breaker. **E1** adds the internal path cost to the seed metric, so different routers can prefer different ASBRs — use E1 when multiple redistribution points exist and you want optimal exit selection.
- `routemap` — always filter redistribution; uncontrolled BGP→OSPF import can flood the LSDB with a full table and melt SPF on small routers.
- `summary-address` — aggregates **external** (Type-5/7) prefixes at the ASBR. Inter-area summarization of internal routes is done at the ABR instead with `config area → config range`. Both reduce LSDB size and, critically, stop distant flapping prefixes from triggering SPF everywhere.

#### OSPF.6 Verification Commands

| Command | Explanation |
|---|---|
| `get router info ospf neighbor` | Adjacency table: neighbor RID, priority, FSM state, dead-timer countdown, and the interface. The go-to health check. |
| `get router info ospf status` | Process overview: RID, ASBR/ABR role, areas attached, SPF run count/last run, LSA counts. |
| `get router info ospf interface <name>` | Per-interface: area, network type, cost, DR/BDR identity, timers, neighbor count. |
| `get router info ospf database brief` | LSDB contents by LSA type — verify what is actually being flooded. |
| `get router info routing-table ospf` | Only the OSPF-derived routes in the RIB (look for `O`, `O IA`, `O E1/E2`, `O N1/N2`). |
| `diagnose ip router ospf all enable` + `diagnose ip router ospf level info` + `diagnose debug enable` | OSPF daemon debug (hello exchange, DD negotiation, SPF). Disable with `diagnose debug disable` and `diagnose ip router ospf all disable`. |
| `execute router clear ospf process` | Restart the OSPF process — full adjacency/LSDB rebuild (disruptive; last resort). |

**Example — `get router info ospf neighbor`:**

```text
OSPF process 0, VRF 0:
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.255.2    250   Full/Backup     00:00:36    10.11.101.2     port1
10.255.255.3      1   Full/DR         00:00:38    10.11.102.3     port2
10.255.255.4      1   Full/ -         00:00:33    10.40.0.2       Agg1
```

- `State Full` — LSDBs are synchronized; this is the only fully healthy end-state for a needed adjacency. The suffix is the **neighbor's** DR role on that segment: `/DR`, `/Backup`, `/DROther`, or `/ -` on point-to-point links (no election).
- `2-Way/DROther` on a broadcast segment is **normal** between two non-DR routers — not a fault.
- `Pri` — the neighbor's advertised DR priority; explains current/next elections.
- `Dead Time` counting down and resetting ≈ every hello-interval = Hellos flowing correctly.
- Failure signatures: stuck **Init** → neighbor doesn't list us in its Hellos (auth/key mismatch, one-way traffic, ACL); stuck **ExStart/Exchange** → MTU mismatch (see `mtu-ignore`); neighbor absent entirely → subnet/area/timer/network-type mismatch or interface not matched by any `network` statement.

**Example — `get router info routing-table ospf`:**

```text
O*E2    0.0.0.0/0 [110/10] via 10.11.102.3, port2, 01:09:48
O       10.11.103.0/24 [110/2] via 10.11.102.3, port2, 00:54:49
                       [110/2] via 10.11.101.2, port1, 00:54:49
O E2    192.168.160.0/24 [110/10] via 10.11.102.3, port2, 01:45:21
```

- `[110/2]` — `[administrative distance / OSPF cost]`. The cost is the summed outgoing-interface costs computed by SPF.
- Two next-hops under one prefix — equal-cost paths installed as **ECMP**.
- `O*E2 0.0.0.0/0` — the default originated by the ASBR via `default-information-originate`; `E2` marks it (like all default-mode redistributed routes) as an external whose metric stayed fixed (`10`) regardless of distance from the ASBR — the E2 behavior described above.
- Route codes decode the LSA machinery: `O` intra-area (Type-1/2), `O IA` inter-area (Type-3 from an ABR), `O E1/E2` external (Type-5 from an ASBR), `O N1/N2` NSSA-external (Type-7).

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