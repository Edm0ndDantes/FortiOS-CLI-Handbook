# High Availability

## Active-Passive FGCP Cluster

```text linenums="1"
config system ha
    set group-id 21
    set group-name "EDGE-HA"
    set mode a-p
    set password <ha-secret>
    set hbdev "ha1" 200 "ha2" 100
    set session-pickup enable
    set session-pickup-connectionless enable
    set override disable
    set priority 200
    set monitor "wan1" "internal1"
    set pingserver-monitor-interface "wan1"
    set pingserver-failover-threshold 5
    set ha-mgmt-status enable
    config ha-mgmt-interfaces
        edit 1
            set interface "mgmt"
            set gateway 10.99.0.254
        next
    end
end
```

- `group-id` — must match on both members and be **unique per L2 domain** (it seeds the virtual MACs; duplicate group-ids on the same segment cause MAC collisions).
- `mode a-p` — active-passive (one member forwards). `a-a` load-balances sessions but complicates troubleshooting; a-p is the sane default.
- `hbdev` — heartbeat interfaces with priorities. Use **two** dedicated, directly-cabled links.
- `session-pickup` — sync TCP sessions so failover is (mostly) hitless; `-connectionless` adds UDP/ICMP sync.
- `override disable` — recommended: prevents the higher-priority unit from preempting when it comes back (avoids a second traffic hit). With `override enable`, priority strictly determines the master.
- `priority` — higher wins master election (when override is enabled or on simultaneous boot).
- `monitor` — interfaces whose link-down triggers failover.
- `pingserver-monitor-interface` (remote link monitoring) — fail over when an upstream target stops answering, catching "link up but path dead" failures. Configure targets under `config system link-monitor`.
- `ha-mgmt-interfaces` — reserved per-unit management so you can always reach the **secondary** directly.

**Both members need:** same model, same firmware, same license tier, matching `group-id/group-name/password`. Configuration syncs automatically from primary to secondary (except HA-reserved settings, hostname and per-device items).

| Command | Explanation |
|---|---|
| `get system ha status` | Cluster health: primary/secondary, uptimes, sync state, election reason. |
| `diagnose sys ha checksum show` | Config checksums per member — mismatch = out of sync. |
| `diagnose sys ha checksum recalculate` | Recompute checksums (first step for false desync alarms). |
| `execute ha synchronize start` | Force a config re-sync to the secondary. |
| `execute ha manage <index> <admin>` | Jump from primary CLI into the secondary's CLI over the HA link. |
| `execute ha failover set 1` | **Force a failover** (7.0+; testing). Clear with `execute ha failover unset 1`. Older method: `diagnose sys ha reset-uptime` on the master. |
| `diagnose sys ha history read` | HA event history — why past failovers happened. |

**Example — `get system ha status` (truncated):**

```text linenums="1"
HA Health Status: OK
Model: FortiGate-200F
Mode: HA A-P
Group: 21
Debug: 0
Cluster Uptime: 41 days 3:2:11
Cluster state change time: 2026-06-02 04:12:33
Primary selected using:
    <2026/06/02 04:12:33> FG200F5622000001 is selected as the primary because its override priority is larger than peer member FG200F5622000002.
Configuration Status:
    FG200F5622000001(updated 1 seconds ago): in-sync
    FG200F5622000002(updated 3 seconds ago): in-sync
System Usage stats: ...
Primary     : FGT-EDGE-01 , FG200F5622000001, HA cluster index = 0
Secondary   : FGT-EDGE-02 , FG200F5622000002, HA cluster index = 1
```

- `Configuration Status: in-sync` on both lines is the key health indicator; `out-of-sync` → run the checksum diagnostics.
- The "Primary selected using" lines explain the last election (priority, uptime, monitored-port failure...), which is your failover root-cause in one line.

---