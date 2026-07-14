# Admin Access & AAA

## Local Admin Accounts, Profiles & Trusted Hosts

```text linenums="1"
config system accprofile
    edit "readonly-noc"
        set secfabgrp read
        set ftviewgrp read
        set authgrp read
        set sysgrp read
        set netgrp read
        set loggrp read
        set fwgrp read
        set vpngrp read
        set utmgrp read
    next
end

config system admin
    edit "noc-viewer"
        set accprofile "readonly-noc"
        set trusthost1 10.99.0.0 255.255.255.0
        set trusthost2 10.100.5.10 255.255.255.255
        set two-factor fortitoken
        set fortitoken "FTKMOB..."
        set password <strong-password>
    next
end
```

- `accprofile` — RBAC profile. Each `*grp` controls read/read-write/none access to a feature area. Build least-privilege profiles instead of handing out `super_admin`.
- `trusthostN` — up to 10 source subnets from which this admin may log in. Any login attempt from outside these is dropped **before** authentication. Always set these.
- `two-factor fortitoken` — enforce 2FA with a FortiToken (mobile or hard token). Alternatives: `email`, `sms`.
- Never delete the default `admin` until a replacement `super_admin` account with trusted hosts and 2FA is verified working.

## RADIUS Authentication for Admins

```text linenums="1"
config user radius
    edit "ISE-RADIUS"
        set server "10.50.1.10"
        set secret <shared-secret>
        set auth-type pap
        set nas-ip 10.0.0.1
        set radius-port 1812
    next
end

config user group
    edit "ADM-NetOps"
        set member "ISE-RADIUS"
        config match
            edit 1
                set server-name "ISE-RADIUS"
                set group-name "NetOps-Admins"
            next
        end
    next
end

config system admin
    edit "netops-wildcard"
        set remote-auth enable
        set remote-group "ADM-NetOps"
        set accprofile "prof_admin"
        set wildcard enable
        set trusthost1 10.99.0.0 255.255.255.0
    next
end
```

- `user radius` — defines the RADIUS server. `auth-type` can be `pap`, `chap`, `ms_chap`, `ms_chap_v2`, or `auto`.
- `nas-ip` — the NAS-IP-Address attribute sent to RADIUS; must often match what the RADIUS server expects.
- `user group` with `config match` — maps a RADIUS-returned group attribute (typically Fortinet-Group-Name VSA or filter-Id) to a FortiGate group.
- `wildcard enable` — one admin entry matches **any** username validated by RADIUS in that group; no need to pre-create each admin locally.
- `remote-auth` + `remote-group` — authenticate against RADIUS and authorize via group membership.

## TACACS+ Authentication for Admins

```text linenums="1"
config user tacacs+
    edit "TAC-SRV"
        set server "10.50.1.20"
        set key <shared-secret>
        set authen-type auto
        set authorization enable
    next
end

config user group
    edit "ADM-TACACS"
        set member "TAC-SRV"
    next
end

config system admin
    edit "tacacs-wildcard"
        set remote-auth enable
        set remote-group "ADM-TACACS"
        set wildcard enable
        set accprofile "super_admin"
        set trusthost1 10.99.0.0 255.255.255.0
    next
end
```

- `authen-type auto` — tries PAP, then CHAP, then ASCII.
- `authorization enable` — allows TACACS+ to return the `admin_prof` attribute, dynamically assigning the access profile per user (central RBAC).
- The rest mirrors the RADIUS wildcard admin pattern.

## Management Access on Interfaces

```text linenums="1"
config system interface
    edit "wan1"
        set allowaccess ping
    next
    edit "mgmt"
        set allowaccess ping https ssh snmp
        set dedicated-to management
    next
end
```

- `allowaccess` — the exhaustive list of management protocols allowed **to** the interface IP. Anything not listed is refused. Never enable `http` or `telnet`; keep WAN interfaces to `ping` at most (or nothing).
- `dedicated-to management` — makes the port a pure out-of-band management interface (own routing, excluded from policies).

| Command | Explanation |
|---|---|
| `get system admin list` | Show currently logged-in administrators (source IP, method). |
| `diagnose sys admin-session list` | Detailed view of active admin sessions. |
| `execute disconnect-admin-session <index>` | Kick a specific admin session. |
| `config system password-policy` | Enforce length/complexity/expiry for admin passwords (see hardening section). |

---