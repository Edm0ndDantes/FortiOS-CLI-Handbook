# System Basics

## Hostname & Timezone

```text
config system global
    set hostname FGT-EDGE-01
    set timezone "Europe/Athens"        # FortiOS 7.4+: named timezones
    set admintimeout 10
    set gui-theme mariner
end
```

- `hostname` — device name shown in prompt, SNMP, HA, and logs.
- `timezone` — on FortiOS ≤ 7.2 this is a numeric index (use `set timezone ?` to list; e.g. `26` = GMT+2). From 7.4 named strings are supported.
- `admintimeout` — idle timeout for admin sessions in minutes (1–480). Keep low (5–10) for security.
- `gui-theme` — cosmetic only; harmless to leave default.

## DNS (System Resolver)

```text
config system dns
    set primary 96.45.45.45
    set secondary 96.45.46.46
    set protocol dot                    # DNS over TLS
    set server-hostname "globalsdns.fortinet.net"
    set domain "corp.example.com"
    set source-ip 10.0.0.1
end
```

- `primary` / `secondary` — resolvers the FortiGate itself uses (FortiGuard, updates, logging by FQDN, admin lookups). Defaults are FortiGuard DNS.
- `protocol` — `cleartext`, `dot` (DNS over TLS), or `doh`. `dot`/`doh` protect DNS privacy/integrity; requires `server-hostname` for certificate validation.
- `domain` — local domain suffix appended to unqualified lookups.
- `source-ip` — force DNS queries to originate from a specific interface IP (useful with VRFs, loopbacks, or strict upstream ACLs).

**DNS server / forwarder on an interface (FortiGate answering client DNS):**

```text
config system dns-database
    edit "corp.example.com"
        set domain "corp.example.com"
        set type primary
        set view shadow
        config dns-entry
            edit 1
                set hostname "srv1"
                set ip 10.10.10.5
            next
        end
    next
end

config system dns-server
    edit "internal1"
        set mode forward-only          # or "recursive", "non-recursive"
    next
end
```

- `dns-database` — defines a local DNS zone. `view shadow` = only answers internal clients; `public` = answers anyone reaching that interface.
- `dns-server` — enables DNS service on interface `internal1`. `forward-only` relays queries to the system DNS servers; `recursive` serves local zones and recursively resolves the rest.

| Command | Explanation |
|---|---|
| `execute nslookup name www.fortinet.com` | Test resolution from the FortiGate itself (uses the configured DNS lookup path). Older builds: `execute nslookup` interactive. |
| `diagnose test application dnsproxy 6` | Show DNS proxy statistics/cache state. |
| `diagnose test application dnsproxy 7` | Dump the DNS cache. |

## NTP

```text
config system ntp
    set ntpsync enable
    set type custom
    set syncinterval 60
    config ntpserver
        edit 1
            set server "0.pool.ntp.org"
        next
        edit 2
            set server "1.pool.ntp.org"
            set ntpv3 disable
        next
    end
    set server-mode enable
    set interface "internal1"
end
```

- `ntpsync enable` — turn on NTP time synchronization. Accurate time is essential for certificates, IPsec, logs, and FortiGuard.
- `type custom` — use your own servers instead of FortiGuard NTP (`fortiguard`).
- `syncinterval` — minutes between syncs (1–1440).
- `ntpv3 disable` — use NTPv4 (default); enable v3 only for legacy servers.
- `server-mode enable` + `interface` — the FortiGate acts as an NTP **server** for internal clients on the listed interfaces. Only enable on trusted internal interfaces.

| Command | Explanation |
|---|---|
| `execute time` | Show or set current system time. |
| `execute date` | Show or set current date. |
| `diagnose sys ntp status` | Show NTP sync state, selected server, stratum, offset. Key check when time-based issues occur (VPN, SSL). |

---