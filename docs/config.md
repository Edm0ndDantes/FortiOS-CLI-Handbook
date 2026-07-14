# Configuration Backup & Restore

| Command | Explanation |
|---|---|
| `execute backup config tftp <filename> <tftp-ip>` | Plaintext config backup to TFTP. |
| `execute backup config tftp <filename> <tftp-ip> <password>` | **Encrypted** backup (also encrypts secrets portably — needed to restore on a different unit with usable passwords/PSKs). |
| `execute backup config ftp <file> <ftp-ip> <user> <pass>` | Backup via FTP; `sftp` variant available on newer builds. |
| `execute backup config usb <filename>` | Backup to USB stick (FAT32). |
| `execute backup config flash <comment>` | Save a revision to local flash (`config system global → set revision-backup-on-logout enable` automates this). |
| `execute revision list config` | List locally stored config revisions. |
| `execute restore config tftp <filename> <tftp-ip>` | Restore config from TFTP — device reboots and applies it. |
| `execute restore config usb <filename>` | Restore from USB. |
| `execute backup full-config tftp <file> <ip>` | Backup including **default values** (documentation/diff use; restore uses normal backups). |
| `show` → copy/paste | For small scopes, `show` output is itself replayable CLI — handy for migrating objects between boxes. |

- Backups are tied to model and (loosely) firmware version — restoring across major versions works only along supported paths; restoring to a different model fails.
- Without an encryption password, secret fields are hashed per-device and won't transplant to different hardware. Use an encrypted backup for replacements/RMA restores.
- HA: back up from the **primary**; the config covers the cluster (HA-reserved mgmt settings are per-device).
- Automate: `config system auto-script` can schedule `execute backup config ...` on an interval, or pull backups via the REST API (`/api/v2/monitor/system/config/backup`).

```text
config system auto-script
    edit "nightly-backup"
        set interval 86400
        set repeat 0
        set script "execute backup config tftp fgt-%%D.conf 10.99.0.50"
    next
end
```

- `interval` in seconds, `repeat 0` = forever. The script body is raw CLI executed as written.

---