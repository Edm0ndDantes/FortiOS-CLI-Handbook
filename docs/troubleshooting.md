# Troubleshooting Guide

## General Method

1. **Define the flow:** src IP/port → dst IP/port/protocol, ingress interface.
2. **Route check:** `get router info routing-table details <dst>` — is there a route, and out the expected interface/VRF?
3. **Policy check:** `diagnose firewall iprope lookup ...` — which policy matches?
4. **Session check:** does a session establish? What does it show?
5. **Packet check:** sniffer — do packets arrive/leave?
6. **Flow debug:** ask the kernel *why* it dropped the packet.

## Connectivity Basics

| Command | Explanation |
|---|---|
| `execute ping 8.8.8.8` | Basic ICMP test from the FortiGate. |
| `execute ping-options source 10.10.10.1` | Set source IP (or `interface <name>`) before pinging — test from a specific segment/VRF. |
| `execute ping-options reset` | Reset ping options to defaults. |
| `execute traceroute 8.8.8.8` / `execute tracert-options ...` | Path tracing, with source options like ping. |
| `execute telnet <ip> <port>` | Quick TCP port reachability test from the box. |

## Packet Sniffer

```text linenums="1"
diagnose sniffer packet <interface> '<filter>' <verbosity> <count> <timestamp>
diagnose sniffer packet any 'host 10.20.0.5 and port 443' 4 100 l
```

- Interface `any` captures on all interfaces (verbosity 4+ shows which).
- Filter is BPF-like: `host`, `net`, `port`, `proto`, `and/or/not`, e.g. `'src 10.1.1.1 and dst port 53'`, `'icmp'`, `'proto 47'`.
- Verbosity: `1` headers, `3` headers+payload hex, `4` headers **with interface name** (the workhorse), `6` = 4 + hex dump (convertible to pcap with Fortinet's script).
- `l` timestamp = local clock. Stop with Ctrl-C.

**Example — verbosity 4:**

```text linenums="1"
interfaces=[any]
filters=[host 10.20.0.5 and port 443]
2.184936 vlan100-users in 10.100.0.23.51512 -> 10.20.0.5.443: syn 3067915582
2.185021 S2S-BRANCH1 out 10.100.0.23.51512 -> 10.20.0.5.443: syn 3067915582
2.212998 S2S-BRANCH1 in 10.20.0.5.443 -> 10.100.0.23.51512: syn 2135437821 ack 3067915583
2.213042 vlan100-users out 10.20.0.5.443 -> 10.100.0.23.51512: syn 2135437821 ack 3067915583
```

- Each packet appears twice (ingress `in`, egress `out`) — this instantly shows routing decisions and NAT rewrites.
- SYN goes out but no SYN-ACK returns → problem beyond the FortiGate (far-end host/firewall/return routing).
- SYN seen `in` but never `out` → the FortiGate dropped it → go to flow debug.

## Flow Debug (why was it dropped?)

```text linenums="1"
diagnose debug reset
diagnose debug flow filter clear
diagnose debug flow filter addr 10.20.0.5
diagnose debug flow filter port 443
diagnose debug flow show function-name enable
diagnose debug flow trace start 20
diagnose debug enable
# ...reproduce the traffic...
diagnose debug disable
diagnose debug reset
```

- Filters narrow to the flow of interest (addr/saddr/daddr/port/proto).
- `trace start 20` — trace the next 20 packets matching the filter.
- **Always** `debug disable` + `reset` afterwards; debug output is CPU-expensive.

**Example output & interpretation:**

```text linenums="1"
id=65308 trace_id=7 func=print_pkt_detail line=5892 msg="vd-root:0 received a packet(proto=6, 10.100.0.23:51512->10.20.0.5:443) tun_id=0.0.0.0 from vlan100-users. flag [S], seq 3067915582"
id=65308 trace_id=7 func=init_ip_session_common line=6077 msg="allocate a new session-00a1b2c3"
id=65308 trace_id=7 func=vf_ip_route_input_common line=2605 msg="find a route: flag=04000000 gw-10.254.1.2 via S2S-BRANCH1"
id=65308 trace_id=7 func=fw_forward_handler line=990 msg="Denied by forward policy check (policy 0)"
```

- Line 1: packet received, 5-tuple, ingress interface.
- Line 3: routing decision (gateway + egress interface) — wrong interface here = routing problem, not policy.
- Line 4: the verdict. `Denied by forward policy check (policy 0)` = hit the **implicit deny** → no policy matches this src/dst-intf + address + service combination. A specific policy ID = that policy denied or a UTM profile blocked it.
- Other classic verdicts: `reverse path check fail, drop` (asymmetric routing / RPF — fix routing or relax `set strict-src-check`), `iprope_in_check() check failed, drop` (local-in: management/VIP/local policy), `no session matched` (reply arrived with no session — asymmetry or timeout).

## Session Table

```text linenums="1"
diagnose sys session filter clear
diagnose sys session filter dst 10.20.0.5
diagnose sys session filter dport 443
diagnose sys session list
diagnose sys session clear      # WARNING: with no filter set this clears ALL sessions
```

**Example session (truncated):**

```text linenums="1"
session info: proto=6 proto_state=01 duration=94 expire=3506 timeout=3600 flags=00000000
origin-shaper=SH-Voice prio=1 ...
state=log may_dirty npu
statistic(bytes/packets/allow_err): org=10932/89/1 reply=482344/341/1 tuples=2
orgin->sink: org pre->post, reply pre->post dev=23->45/45->23 gwy=10.254.1.2/10.100.0.23
hook=post dir=org act=snat 10.100.0.23:51512->10.20.0.5:443(203.0.113.5:51512)
hook=pre dir=reply act=dnat 10.20.0.5:443->203.0.113.5:51512(10.100.0.23:51512)
policy_id=40 auth_info=0 chk_client_info=0 vd=0
```

- `proto_state=01` for TCP = ESTABLISHED (first digit reply-dir state, second origin; `05` = SYN sent, no reply yet — the classic "no return traffic" signature).
- `statistic` — bytes/packets in each direction; zero reply bytes = one-way traffic.
- `hook/act` lines show exactly what NAT was applied.
- `policy_id` — which policy admitted the session.
- `npu` flag — session is hardware-offloaded (a sniffer won't see offloaded packets mid-session; disable offload on the policy — `set auto-asic-offload disable` — when packet-level debugging).

## Resource / System Health

| Command | Explanation |
|---|---|
| `get system performance status` | CPU (per core), memory, session count/setup rate, uptime, network throughput. |
| `diagnose sys top 2 20` | Top processes, refresh 2 s. High `ipsengine`/`wad` = inspection load. Press `q` to quit, `m` sort by memory, `c` by CPU. |
| `diagnose sys session stat` | Session table totals, setup rate, clash/failure counters. |
| `diagnose hardware sysinfo memory` | Detailed memory breakdown. |
| `diagnose sys mpstat 2` | Per-CPU utilization live. |
| `get system session status` | Current session count vs table size. |
| `diagnose debug crashlog read` | Daemon crash history — check after any instability. |
| `execute tac report` | Generate the full TAC diagnostic bundle for Fortinet support. |

**Conserve mode:** when free memory drops below thresholds the box enters conserve mode (new sessions/AV inspection restricted). Detect with:

```text linenums="1"
diagnose hardware sysinfo conserve
```

```text linenums="1"
memory conserve mode:   off
total RAM:              8192 MB
memory used:            4211 MB  51% of total RAM
memory freeable:        1298 MB  15% of total RAM
memory used + freeable threshold extreme: 7783 MB  95% of total RAM
memory used threshold red:                7209 MB  88% of total RAM
memory used threshold green:              6717 MB  82% of total RAM
```

- `red` threshold entry → conserve mode on; `green` → exit. If it triggers, find the consumer with `diagnose sys top` (commonly `wad` with deep inspection) and reduce load or restart the offending daemon (`diagnose test application wad 99`).

## Quick Scenario Index

| Symptom | First commands |
|---|---|
| Traffic blocked, unclear why | `diagnose firewall iprope lookup`, then flow debug (17.4) |
| Works one way only / asymmetric | `diagnose sys session list` (check proto_state), sniffer both interfaces, look for `reverse path check fail` in flow debug |
| IPsec won't come up | `diagnose vpn ike gateway list`, then `diagnose debug application ike -1` — read which proposal/selector the peer rejects |
| IPsec up, no traffic | `diagnose vpn tunnel list` (counters), route check to the tunnel, both-direction policies, blackhole distance |
| OSPF neighbor missing | `get router info ospf interface`, check area/type/timers/auth; MTU if stuck ExStart |
| BGP session down | `get router info bgp summary`, `execute ping-options source` + ping peer, check TCP 179 with sniffer |
| Slow throughput | `get system performance status`, `diagnose sys top`, check `npu` flag on sessions, shaper drop counters (13) |
| HA out of sync | `diagnose sys ha checksum show`, `execute ha synchronize start` |
| DNS oddities | `execute nslookup`, `diagnose test application dnsproxy 6/7` |
| High memory / conserve | `diagnose hardware sysinfo conserve`, `diagnose sys top` |
| Intermittent drops on an interface | `diagnose hardware deviceinfo nic <port>` (CRC/collisions), `diagnose netlink interface list` |
| Admin locked out (source blocked) | Console access → check `trusthostN`, `admin-lockout` state; `diagnose firewall iprope list 100011` shows local-in rules |

---

*End of cheatsheet. Verify syntax against your exact FortiOS build (`get system status`) — attribute names occasionally shift between major releases; `?` and `tree` are authoritative on-box.*
