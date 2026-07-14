# CLI Basics & Navigation

| Command | Explanation |
|---|---|
| `config <path>` | Enter a configuration table (e.g. `config system interface`). |
| `edit <name/id>` | Create or open an entry inside a table. Creates it if it doesn't exist. |
| `set <attribute> <value>` | Set an attribute inside an entry. |
| `unset <attribute>` | Revert an attribute to its default value. |
| `append <attribute> <value>` | Add a value to a multi-value attribute (e.g. add a member to a list) without retyping the whole list. |
| `select <attribute> <value>` | Remove a value from a multi-value attribute (FortiOS 7.x: `unselect`). |
| `next` | Save the current entry and move back to the table level (to edit another entry). |
| `end` | Save all changes and exit the configuration table completely. |
| `abort` | Exit the config table **without saving** changes in the current scope. |
| `delete <name/id>` | Delete an entry from a table. |
| `purge` | Delete **all** entries in a table. Use with extreme caution. |
| `show` | Display the non-default configuration of the current scope. |
| `show full-configuration` | Display the configuration including all default values. |
| `get` | At config level: show all attributes of the current entry. At exec level: show status/system info (e.g. `get system status`). |
| `get system status` | Show firmware version, serial number, hostname, HA mode, license state. First command to run on any box. |
| `tree` | Show the full command tree available under the current scope. |
| `?` | Context help: list available commands/options at the cursor position. |
| `execute <command>` | Run operational commands (ping, reboot, backup, etc.). |
| `diagnose <command>` | Advanced debugging and diagnostics. |
| `alias <name>` | Configure a CLI alias for a long command (`config system alias`). |
| `execute cli check-template-status` | Verify config sync/template status (FortiManager-managed devices). |
| `grep` filter: `show \| grep -f <string>` | Filter output. `-f` prints the surrounding config block, not just the matching line — extremely useful. |
| `get system performance status` | Quick CPU, memory, session, and uptime overview. |

**Useful output modifiers:**

```text
show full-configuration | grep -f ospf     # show blocks containing "ospf"
get router info routing-table all | grep 10.10.
diagnose sys session list | grep -c ""      # count lines (session count)
```

- `| grep <pattern>` — filter lines matching pattern.
- `| grep -f <pattern>` — print the whole enclosing config block (FortiOS-specific and very handy).
- `| grep -v <pattern>` — invert match.
- `| grep -c <pattern>` — count matches.

---