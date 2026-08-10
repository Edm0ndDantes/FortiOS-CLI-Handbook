
# Dynamic Routing

### OSPF

**Theory recap in one paragraph:** OSPF is a link-state IGP (RFC 2328). Every router floods Link-State Advertisements (LSAs) describing its links; all routers in an area build an identical Link-State Database (LSDB) and independently run Dijkstra's SPF algorithm to compute shortest paths, using **cost** as the metric. Routers discover each other with multicast **Hello** packets (224.0.0.5), progress through an adjacency state machine (Down → Init → 2-Way → ExStart → Exchange → Loading → **Full**), and on multi-access segments elect a **Designated Router (DR)** and **Backup DR (BDR)** to reduce adjacency count from O(n²) to O(n). Areas bound LSA flooding; area 0.0.0.0 is the **backbone** that all other areas must attach to. Routers injecting external routes (e.g. redistributed BGP/static) are **ASBRs**; routers joining areas are **ABRs**.

#### OSPF.1 Process-Level Configuration

```text linenums="1"
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

```text linenums="1"
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

```text linenums="1"
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

```text linenums="1"
config router ospf
    set passive-interface "VLAN100-Users"
end
```

- Passive interfaces suppress Hellos (no neighbors possible) but the prefix stays in the Router LSA — the standard way to advertise stub networks while eliminating the attack/misconfig surface of unnecessary adjacencies.

#### OSPF.4 Per-Interface Parameters (`ospf-interface`)

```text linenums="1"
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

```text linenums="1"
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

```text linenums="1"
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

```text linenums="1"
O*E2    0.0.0.0/0 [110/10] via 10.11.102.3, port2, 01:09:48
O       10.11.103.0/24 [110/2] via 10.11.102.3, port2, 00:54:49
                       [110/2] via 10.11.101.2, port1, 00:54:49
O E2    192.168.160.0/24 [110/10] via 10.11.102.3, port2, 01:45:21
```

- `[110/2]` — `[administrative distance / OSPF cost]`. The cost is the summed outgoing-interface costs computed by SPF.
- Two next-hops under one prefix — equal-cost paths installed as **ECMP**.
- `O*E2 0.0.0.0/0` — the default originated by the ASBR via `default-information-originate`; `E2` marks it (like all default-mode redistributed routes) as an external whose metric stayed fixed (`10`) regardless of distance from the ASBR — the E2 behavior described above.
- Route codes decode the LSA machinery: `O` intra-area (Type-1/2), `O IA` inter-area (Type-3 from an ABR), `O E1/E2` external (Type-5 from an ASBR), `O N1/N2` NSSA-external (Type-7).

### BGP

**Theory recap in one paragraph:** BGP-4 (RFC 4271) is a **path-vector** exterior gateway protocol. Unlike link-state IGPs, routers don't build a topology map — they exchange full routes ("NLRI") with **path attributes** (AS_PATH, NEXT_HOP, LOCAL_PREF, MED, communities...) over a plain **TCP session on port 179**, and apply *policy* to choose and propagate paths. Loop prevention is the AS_PATH itself: a router rejects any route already containing its own AS number. Sessions between different ASes are **eBGP** (policy boundary, next-hop rewritten, TTL=1 by default); sessions inside one AS are **iBGP** (next-hop unchanged, and — because iBGP-learned routes are never re-advertised to other iBGP peers — you need a **full mesh or route reflectors**). There are no periodic re-floods: after the initial table exchange only **incremental UPDATEs** flow, with KEEPALIVEs (default 60 s) guarding a hold timer (default 180 s). A session walks the FSM Idle → Connect/Active → OpenSent → OpenConfirm → **Established**; only in Established are routes exchanged. When multiple paths to a prefix exist, the **best-path algorithm** picks one, in order: highest weight (Fortinet/Cisco-local) → highest LOCAL_PREF → locally originated → shortest AS_PATH → lowest origin (IGP<EGP<Incomplete) → lowest MED → eBGP over iBGP → lowest IGP cost to next-hop → oldest → lowest RID.

#### BGP.1 Process-Level Configuration

```text linenums="1"
config router bgp
    set as 65010
    set router-id 10.255.255.1
    set keepalive-timer 60                      # KEEPALIVE interval (s)
    set holdtime-timer 180                      # Session declared dead after this silence
    set ebgp-multipath enable                   # ECMP across equal eBGP paths
    set ibgp-multipath disable
    set network-import-check enable             # Only advertise 'network' prefixes present in RIB
    set log-neighbour-changes enable            # Log session up/down transitions
    set graceful-restart enable                 # RFC 4724 hitless restart / HA failover
    set scan-time 60                            # Next-hop validation scan interval
    set always-compare-med disable
    set deterministic-med enable
end
```

- `as` — this router's Autonomous System number. BGP theory: the ASN prepended into AS_PATH on every eBGP advertisement, and the value that determines session *type* — a neighbor whose `remote-as` equals ours is iBGP, anything else is eBGP. Use 64512–65534 / 4200000000+ for private ASNs.
- `router-id` — the 32-bit BGP Identifier exchanged in the OPEN message; it's the final tie-breaker of the best-path algorithm and must be unique between peers (duplicate RIDs refuse to peer). Set it explicitly to the loopback, same discipline as OSPF.
- `keepalive-timer` / `holdtime-timer` — the liveness machinery. Hold time is **negotiated down** to the lower of the two peers' OPEN values; keepalive is then typically ⅓ of it. Lowering these (e.g. 20/60 or with BFD instead) is the main lever for failover speed on static peerings.
- `ebgp-multipath` — installs multiple equal candidates (same weight/local-pref/AS_PATH length/MED) as ECMP instead of electing a single best path. Required for dual-ISP or dual-hub load sharing; without it BGP always installs exactly one path.
- `network-import-check` — RFC-faithful behavior: a `config network` prefix is only advertised if a matching route exists in the RIB (BGP theory: you should only originate reachability you actually have). Disable only for deliberate advertisement of not-yet-routed aggregates.
- `graceful-restart` — RFC 4724: peers keep forwarding on our stale routes while the BGP process restarts (crucial on HA clusters, where failover restarts bgpd on the new primary).
- `scan-time` — how often BGP re-validates that each route's NEXT_HOP is still resolvable in the IGP/static RIB; recursive resolution failing = route withdrawn.
- `deterministic-med` / `always-compare-med` — MED is only comparable between paths from the **same** neighboring AS by default (it's a hint *that* AS set). `deterministic-med` groups paths per-AS before comparing (sane, enable); `always-compare-med` compares MED across different ASes (only if your design defines MED globally).

#### BGP.2 Neighbors

```text linenums="1"
config router bgp
    config neighbor
        edit "203.0.113.1"                      # eBGP: ISP transit
            set remote-as 65001
            set description "ISP-A transit"
            set password <md5-secret>           # TCP MD5 (RFC 2385) session protection
            set prefix-list-in "PL-DEFAULT-ONLY"
            set route-map-out "RM-ANNOUNCE-PA"
            set maximum-prefix 100
            set maximum-prefix-threshold 80
            set soft-reconfiguration enable
            set bfd enable
        next
        edit "10.255.255.2"                     # iBGP: core peer via loopbacks
            set remote-as 65010
            set update-source "Lo0"
            set next-hop-self enable
            set soft-reconfiguration enable
        next
    end
end
```

- `remote-as` — declares the expected peer AS; the OPEN message is checked against it (mismatch → NOTIFICATION, session torn down). Equal to local `as` = iBGP, different = eBGP — this single line changes next-hop handling, TTL, and re-advertisement rules per BGP theory.
- `update-source` — which interface/IP sources the TCP/179 connection. Theory: the packets must arrive from exactly the IP the peer has configured as its neighbor, so loopback-to-loopback iBGP requires both sides to source from the loopback **and** have IGP/static reachability to it. This is also the standard pattern for BGP over IPsec/GRE tunnels (peer on the tunnel or loopback IPs) so the session survives any single physical path via the IGP. Fortinet's basic example uses exactly this for iBGP across a VPN.
- `next-hop-self` — rewrites NEXT_HOP to our own address when advertising to this iBGP peer. Theory: iBGP does **not** change next-hop, so routes learned from eBGP carry the external next-hop (e.g. the ISP's IP) into the AS; interior routers can't resolve it and the route stays inactive. `next-hop-self` on border routers is the classic fix (alternative: redistribute the external subnet into the IGP).
- `ebgp-enforce-multihop` + `set ebgp-multihop-ttl <n>` — eBGP sends TCP with **TTL 1** by default (peers assumed directly connected — a built-in safety). Loopback-based or multi-hop eBGP needs the TTL raised.
- `password` — TCP MD5 authentication: every segment is signed, killing blind RST/ hijack attacks on the long-lived session. Always set on eBGP over shared/exposed media.
- `soft-reconfiguration enable` — stores the peer's routes **pre-policy** (Adj-RIB-In) so `execute router clear bgp ... soft in` re-applies changed filters without bouncing the session. Costs memory; modern peers also negotiate the route-refresh capability which achieves the same without storage.
- `maximum-prefix` (+ threshold %) — protection against a peer leaking a full table into you (classic outage cause): session is dropped past the limit.
- `bfd enable` — sub-second failure detection decoupled from BGP timers; the BFD session (configured under `config system bfd` / interface) signals the routing protocol on path loss in ~150–300 ms.
- `route-reflector-client enable` (on the RR, per iBGP neighbor) — relaxes the iBGP re-advertisement prohibition: the RR reflects routes between clients, replacing the full mesh; ORIGINATOR_ID/CLUSTER_LIST attributes take over loop prevention.
- `allowas-in` / `remove-private-as` — AS_PATH manipulation: accept routes containing our own ASN (hub-and-spoke re-advertisement through a provider) / strip private ASNs before announcing upstream.

**Dynamic peers (hub side of ADVPN / dial-up overlays):**

```text linenums="1"
config router bgp
    config neighbor-group
        edit "SPOKES"
            set remote-as 65010
            set next-hop-self enable
            set advertisement-interval 1
            set link-down-failover enable
        next
    end
    config neighbor-range
        edit 1
            set prefix 10.254.0.0 255.255.0.0
            set neighbor-group "SPOKES"
            set max-neighbor-num 200
        next
    end
end
```

- `neighbor-group` + `neighbor-range` — accept sessions from **any** source inside the prefix and apply the group template: no per-spoke configuration, the scaling pattern for large overlays.

#### BGP.3 Originating Routes: Networks, Redistribution, Aggregation

```text linenums="1"
config router bgp
    config network
        edit 1
            set prefix 192.0.2.0 255.255.255.0
        next
    end
    config redistribute "static"
        set status enable
        set route-map "RM-STATIC-TO-BGP"
    end
    config aggregate-address
        edit 1
            set prefix 10.11.0.0 255.255.0.0
            set summary-only enable             # Suppress the contributing /24s
            set as-set disable
        next
    end
end
```

- `config network` — originates the prefix with **origin code IGP** (best origin in path selection) if it exists in the RIB (see `network-import-check`). This is the clean, intentional way to announce your space.
- `config redistribute` — imports static/connected/OSPF/RIP/ISIS routes, marked **origin Incomplete** (worst origin). Always constrain with a route-map: unfiltered redistribution is the textbook route-leak generator.
- `aggregate-address` — creates a summary when at least one contributing route exists. `summary-only` suppresses the specifics (only the aggregate leaves — smaller global impact); `as-set` preserves the union of contributors' AS_PATHs inside the aggregate for loop-safety when aggregating across ASes.

#### BGP.4 Policy: Prefix-Lists, Route-Maps, and the Best-Path Levers

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
    edit "RM-PREFER-ISPA"
        config rule
            edit 1
                set match-ip-address "PL-ISPA-ROUTES"
                set set-local-preference 200    # AS-wide preference (iBGP-propagated)
            next
            edit 100
                # implicit permit-all tail rule keeps other routes untouched
            next
        end
    next
    edit "RM-DEPREF-BACKUP-OUT"
        config rule
            edit 1
                set set-aspath "65010 65010 65010"   # AS_PATH prepend on advertisements
            next
        end
    next
end
```

- Prefix-lists match on prefix + length (`ge`/`le` give ranges, e.g. `set prefix 10.0.0.0 255.0.0.0`, `set le 24`); the primary matching tool for BGP filtering.
- Route-maps are ordered permit/deny rules with `match` and `set` clauses — BGP's policy engine. The `set` actions map directly onto the best-path algorithm, which is how you steer traffic:
  - `set-local-preference` — **inbound/interior lever**: LOCAL_PREF is compared 2nd, propagates through all iBGP, and defines the whole AS's preferred exit. Higher wins.
  - `set-aspath` (prepending) — **outbound lever**: artificially lengthens AS_PATH on your advertisements so the *rest of the internet* prefers your other link (step 4 of selection). Coarse but universal.
  - `set-metric` (MED) — polite hint to a **directly neighboring AS** about which of your multiple links to prefer inbound. Lower wins; only honored between paths from the same AS (see `deterministic-med`).
  - `set-community` / `match-community` (with `config router community-list`) — arbitrary route tags (`65010:100`); the standard signaling mechanism to ISPs (blackhole, no-export, regional-only) and inside your own policy. Requires `set send-community` standard on the neighbor (default on FortiOS).
  - `set-weight` — Fortinet/Cisco-local attribute, checked **first**, never advertised: per-router override that beats everything.
- Attach with `route-map-in` / `route-map-out` per neighbor. eBGP hygiene: **always** filter both directions — accept only what you expect, announce only your own aggregates.

#### BGP.5 Verification Commands

| Command | Explanation |
|---|---|
| `get router info bgp summary` | Peer table: AS, uptime, FSM state / prefixes received. First stop for session health. |
| `get router info bgp neighbors 203.0.113.1` | Full peer detail: negotiated timers, capabilities, message counters, last NOTIFICATION reason. |
| `get router info bgp neighbors 203.0.113.1 routes` | Routes **accepted** from the peer (post-inbound-policy). |
| `get router info bgp neighbors 203.0.113.1 advertised-routes` | What we **announce** to the peer (post-outbound-policy) — verify filters here. |
| `get router info bgp network 203.0.113.0/24` | The BGP table entry: all paths, their attributes, and which was chosen best and *why*. |
| `get router info routing-table bgp` | Only BGP routes actually installed in the RIB. |
| `execute router clear bgp ip 203.0.113.1 soft in` / `soft out` | Re-apply policy without dropping the TCP session (needs soft-reconfig or route-refresh). |
| `execute router clear bgp ip 203.0.113.1` | Hard reset — tears down and re-establishes the session (disruptive). |
| `diagnose ip router bgp all enable` + `diagnose ip router bgp level info` + `diagnose debug enable` | bgpd debug: OPEN negotiation, UPDATE processing, policy hits. Disable afterwards. |
| `diagnose sniffer packet any 'tcp and port 179' 4` | Watch the session at packet level — the definitive tool when peers can't even reach Established (SYN unanswered, RSTs, wrong source IP). |

**Example — `get router info bgp summary`:**

```text linenums="1"
BGP router identifier 10.255.255.1, local AS number 65010
BGP table version is 14
2 BGP AS-PATH entries
Neighbor        V   AS     MsgRcvd MsgSent  TblVer InQ OutQ Up/Down    State/PfxRcd
203.0.113.1     4   65001   152344  151201      14   0    0  5d02h11m         187
10.255.255.2    4   65010     8812    8801      14   0    0  4d11h02m          42
198.51.100.9    4   65002      121     138       0   0    0  never          Active
```

- A **number** in `State/PfxRcd` means Established — the FSM's only forwarding state — showing how many prefixes survived inbound policy. `0` while Established = session fine, inbound filter eating everything (or peer sending nothing): check `... routes` vs the peer's intent.
- `Active`/`Connect` — the TCP session itself won't establish. Theory: Connect = we're trying the SYN, Active = retrying after failure. Causes: no route to peer, wrong `update-source`, TTL (missing `ebgp-multihop`), TCP/179 filtered, peer not configured. Confirm with the sniffer on port 179.
- `Idle` — not even attempting: admin down (`set shutdown enable`), config error, or max-prefix violation penalty.
- `OpenSent/OpenConfirm` flapping — TCP is fine but OPEN parameters are rejected: AS mismatch, duplicate RID, bad MD5 password, or hold-time expiry (MTU/CPU trouble).
- `Up/Down` resetting frequently with Established = session flaps — check BFD/hold timers, MTU on tunnels, and `get router info bgp neighbors` last-error field.

**Example — `get router info bgp network 0.0.0.0/0` (best-path reasoning):**

```text linenums="1"
BGP routing table entry for 0.0.0.0/0
Paths: (2 available, best #1, table Default-IP-Routing-Table)
  65001
    203.0.113.1 from 203.0.113.1 (203.0.113.99)
      Origin IGP, localpref 200, valid, external, best
      Community: 65010:100
      Last update: Wed Aug  5 09:12:41 2026
  65002 65002 65002 65001
    198.51.100.9 from 198.51.100.9 (198.51.100.77)
      Origin IGP, localpref 100, valid, external
      Last update: Wed Aug  5 09:12:44 2026
```

- Read the tie-break directly against the algorithm: path #1 wins on **localpref 200 vs 100** (step 2), so the comparison never reaches AS_PATH — though the prepended `65002 65002 65002 65001` path would have lost step 4 anyway. This output is where "why is traffic leaving via ISP-A?" gets answered definitively.
- `(203.0.113.99)` — the peer's BGP router-id; `valid` = next-hop resolvable; `external` = learned via eBGP; only paths marked `best` are candidates for the RIB.

**Example — `get router info routing-table bgp`:**

```text linenums="1"
B*      0.0.0.0/0 [20/0] via 203.0.113.1, port1, 5d02h11m
B       10.20.0.0/16 [200/0] via 10.255.255.2 (recursive via 10.254.1.2), S2S-BRANCH1, 4d11h02m
```

- `[20/0]` vs `[200/0]` — FortiOS administrative distance encodes the theory split: **eBGP 20** (trusted like an IGP-external), **iBGP 200** (least trusted — interior reachability should come from the IGP). The second value is the MED.
- `recursive via` — the iBGP route's NEXT_HOP (a loopback) was resolved through another RIB entry (the tunnel route): BGP's recursive next-hop resolution in action, and the thing that breaks (route withdrawn) when the IGP loses the loopback.