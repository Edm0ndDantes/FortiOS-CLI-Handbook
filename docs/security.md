# Device Hardening & Security Best Practices

## Global Hardening

```text
config system global
    set admintimeout 10
    set admin-lockout-threshold 3
    set admin-lockout-duration 300
    set admin-https-redirect enable
    set admin-sport 8443
    set admin-ssh-port 2222
    set admin-ssh-grace-time 30
    set admin-ssh-v1 disable
    set admin-telnet disable
    set admin-concurrent disable
    set strong-crypto enable
    set ssh-hmac-md5 disable
    set ssh-cbc-cipher disable
    set ssh-kex-sha1 disable
    set ssl-static-key-ciphers disable
    set admin-https-ssl-versions tlsv1-2 tlsv1-3
    set autoinstall-config disable
    set autoinstall-image disable
    set gui-firmware-upgrade-warning enable
end
```

- `admin-lockout-*` — brute-force lockout: 3 failures → 300 s ban per source.
- `admin-sport` / `admin-ssh-port` — non-default admin ports reduce scanner noise (not a substitute for trusted hosts).
- `strong-crypto enable` — globally disables weak ciphers/digests (MD5, RC4, static RSA, SSLv3/TLS1.0...) across admin GUI, SSL VPN, LDAPS, etc. **Foundation setting.**
- `admin-telnet disable` and `admin-ssh-v1 disable` — remove plaintext/legacy admin protocols entirely.
- `admin-concurrent disable` — no simultaneous logins with the same account (accountability).
- `autoinstall-* disable` — prevents auto-loading config/firmware from USB at boot (physical attack vector).

```text
config system password-policy
    set status enable
    set apply-to admin-password ipsec-preshared-key
    set minimum-length 14
    set min-lower-case-letter 1
    set min-upper-case-letter 1
    set min-number 1
    set min-non-alphanumeric 1
    set expire-status enable
    set expire-day 180
    set reuse-password disable
end
```

- Enforces complexity for admin passwords **and** IPsec PSKs; `reuse-password disable` blocks reusing the previous password.

## Attack-Surface Reduction

```text
config system interface
    edit "wan1"
        set allowaccess ""            # nothing; or "ping" only if needed
    next
end

config system global
    set admin-server-cert "your-CA-signed-cert"
end

config system auto-install
    set auto-install-config disable
    set auto-install-image disable
end

config system session-ttl
    set default 3600
end
```

- Empty `allowaccess` on untrusted interfaces = no management plane exposure at all. Manage via dedicated mgmt interface/VPN only.
- Replace the built-in self-signed admin cert with a CA-issued one — prevents users normalizing certificate warnings.

**Additional practice checklist (config locations in parentheses):**

- Disable unused features/visibility to shrink the config and GUI (`config system settings → set gui-* disable`).
- Log the implicit deny: `config firewall policy → edit 0 → set logtraffic all` (policy ID 0 is the implicit deny in 7.x; on older builds enable via GUI equivalent `set implicit-log enable` under `config system settings`).
- Enable FortiGuard scheduled updates and check license validity: `execute update-now`, `diagnose autoupdate versions`.
- Send logs off-box: `config log syslogd setting` (set `status enable`, `server`, `mode reliable`, `enc-algorithm high` for TLS syslog).
- SNMPv3 only, with auth+priv: `config system snmp user → set security-level auth-priv → set auth-proto sha256 → set priv-proto aes256`. Delete v1/v2c communities.
- Restrict FortiGuard/DNS/NTP source with `set source-ip` to management/loopback addresses so egress filtering can be strict.
- Enable `config system central-management` only if actually managed by FortiManager; otherwise leave disabled.
- Set login banners (`config system replacemsg admin pre_admin-disclaimer-text`) for legal notice.
- Review `diagnose sys top` and `get system performance status` baselines so anomalies are recognizable.
- Track PSIRT advisories and patch promptly — most exploited FortiOS bugs target exposed management/SSL-VPN daemons; keeping those off untrusted interfaces removes most of the risk.
- Verify trusted hosts on **every** admin: `show system admin | grep -f trusthost`.
- Disable the USB console/auto-install if devices sit in shared racks.

| Command | Explanation |
|---|---|
| `diagnose sys top 2 20` | Live process CPU/memory — spot rogue daemons/load. |
| `get system performance status` | CPU, memory, sessions, network usage snapshot. |
| `diagnose autoupdate versions` | Installed FortiGuard package versions (AV/IPS defs, geodb). |
| `execute update-now` | Force FortiGuard definition update. |
| `diagnose test update info` | FortiGuard connectivity/last update status. |
| `diagnose log test` | Generate test log entries to verify syslog/FAZ delivery. |
| `execute log fortianalyzer test-connectivity` | Verify FortiAnalyzer reachability/registration. |

---