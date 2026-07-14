# Firmware Upgrade Guide

**Golden rules:** follow the official **upgrade path** (Fortinet Docs → Upgrade Path tool) — skipping intermediate versions corrupts config conversions; always back up first; read the release notes for your model; upgrade the **secondary** HA member first (uninterruptible upgrade does this automatically).

## Pre-Upgrade Checklist

| Command | Explanation |
|---|---|
| `get system status` | Record current version/build, serial, license state. |
| `execute backup config tftp pre-upgrade.conf 10.99.0.50` | Full config backup to TFTP before touching anything. |
| `diagnose sys flash list` | Show both firmware partitions and which is active — your rollback lives on the other partition. |
| `get system ha status` | Confirm cluster is healthy and in-sync before an HA upgrade. |
| `execute disk list` / `diagnose hardware deviceinfo disk` | Check disk health (logging disk issues can break upgrades). |

## Upgrade via TFTP

```text
execute restore image tftp FGT_200F-v7.4.4.F-build2662-FORTINET.out 10.99.0.50
```

- Downloads the image, verifies it, writes to the **inactive** partition, and reboots into it. Console access strongly recommended to watch the boot.
- HA cluster: with `uninterruptible-upgrade enable` (default, `config system ha`), the cluster upgrades the secondary first, fails over, then upgrades the former primary — near-zero downtime.

## Upgrade via FortiGuard (no image file handling)

```text
execute upgrade fortiguard list          # list available versions for this model
execute upgrade fortiguard <version-id>  # download+install directly (7.2+ syntax varies: also "execute restore image fortiguard <ver>")
```

- Requires valid support contract and FortiGuard reachability; the box only offers versions on your valid upgrade path.

## Rollback

```text
diagnose sys flash list
execute set-next-reboot {primary | secondary}
execute reboot
```

- The previous firmware (and the config that was running under it) remains on the other flash partition. `set-next-reboot` flips the active partition — fastest clean rollback.
- Note: booting the old partition restores the old **config snapshot** from that partition too; re-apply any config made after the upgrade or restore from backup.

## Post-Upgrade Verification

| Command | Explanation |
|---|---|
| `get system status` | Confirm expected version/build. |
| `get system ha status` | Both members upgraded and in-sync. |
| `get router info routing-table all` | Routing converged as before. |
| `get vpn ipsec tunnel summary` | Tunnels re-established. |
| `diagnose debug crashlog read` | Check for daemon crashes after boot. |
| `diagnose autoupdate versions` | FortiGuard packages still updating on the new build. |

---
