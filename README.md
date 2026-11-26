# ansible-role-adguardhome

Install [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome) - network-wide ads & trackers blocking DNS server.

## Requirements

- Debian/Ubuntu
- Collection: `ansible.utils`

## Supported Distributions

- Debian (bullseye, bookworm)
- Ubuntu (focal, jammy, noble)

Other distributions will fail with an error message.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `adguardhome_version` | `v1.0.0` | Version to install (always exact) |
| `adguardhome_schema_version` | `35` | Config schema version for initial template (fresh install only) |
| `adguardhome_update` | `false` | Force update if installed |
| `adguardhome_install_dir` | `/opt/AdGuardHome` | Installation directory |
| `adguardhome_disable_resolved` | `true` | Disable systemd-resolved |
| `adguardhome_configure_resolv` | `true` | Configure /etc/resolv.conf |
| `adguardhome_fallback_dns` | `[1.1.1.1, 8.8.8.8]` | Fallback DNS servers |
| `adguardhome_deploy_config` | `true` | Deploy initial config (fresh install only) |
| `adguardhome_web_host` | from `wireguard_address` | Web UI listen address |
| `adguardhome_web_port` | `3000` | Web UI port |
| `adguardhome_admin_user` | `admin` | Admin username |
| `adguardhome_admin_pass_hash` | - | Bcrypt password hash |
| `adguardhome_language` | `en` | Interface language |
| `adguardhome_theme` | `auto` | UI theme |
| `adguardhome_dns_bind_hosts` | from `wireguard_address` | DNS listen addresses |
| `adguardhome_dns_port` | `53` | DNS port |
| `adguardhome_upstream_dns` | Cloudflare, Google, Quad9 | Upstream DNS servers |
| `adguardhome_bootstrap_dns` | `[1.1.1.1, 8.8.8.8]` | Bootstrap DNS |
| `adguardhome_dnssec` | `true` | Enable DNSSEC |
| `adguardhome_ratelimit` | `30` | Rate limit (requests/sec per client subnet, 0 = off) |
| `adguardhome_ratelimit_subnet_len_ipv4` | `24` | Client subnet for the rate limit; `32` = per client (VPN peers in one /24 otherwise share it) |
| `adguardhome_anonymize_client_ip` | `false` | Store client IPs truncated in logs and stats |
| `adguardhome_querylog_enabled` | `true` | Query log (per-client domain history, 24h) |
| `adguardhome_filters` | AdGuard, AdAway | DNS filters |
| `adguardhome_user_rules` | `[]` | Custom whitelist rules |
| `adguardhome_sync_filtering` | `false` | Keep `filters` and `user_rules` of an existing install in sync with the inventory |
| `adguardhome_sync_settings` | `false` | Same for rate limit, IP anonymization and query log; query log off also deletes stored log files |

## Version Management

**Important:** Always specify exact version in `defaults/main.yml`.

The template `AdGuardHome.yaml.j2` uses `adguardhome_schema_version` — it is applied **only on fresh installs**.
On updates, AdGuard Home automatically migrates `AdGuardHome.yaml` to the new schema on startup.

### Current version

```yaml
adguardhome_version: "v1.0.0"
adguardhome_schema_version: 35  # used only for fresh installs
```

### Updating to new version

1. Check [AdGuard Home releases](https://github.com/AdguardTeam/AdGuardHome/releases)
2. Update `adguardhome_version` in `defaults/main.yml`
3. If you want fresh installs to use the new schema: update `adguardhome_schema_version` and `templates/AdGuardHome.yaml.j2`
4. Run update — AGH will auto-migrate the existing config on startup

## Updating AdGuard Home

Update replaces only the binary file, keeping configuration and data intact.
This follows the [official manual update procedure](https://github.com/AdguardTeam/AdGuardHome/wiki/FAQ#manual-update).

### What happens during update

1. Download new version archive
2. Stop AdGuard Home service
3. **Backup binary, config and data** to `/opt/AdGuardHome.backup.<version>/`
4. **Save config snapshot** as `/opt/AdGuardHome/AdGuardHome.yaml.<version>` (pre-migration copy)
5. Auto-cleanup old backups (keeps last 3)
6. Extract and replace only the binary
7. Restart service

### What's backed up

Per-version backup directory `/opt/AdGuardHome.backup.<version>/` contains:
- `AdGuardHome` - binary of the old version
- `AdGuardHome.yaml` - config before the new binary starts (pre-schema-migration)
- `data/` - databases, statistics, query logs, downloaded filters

Additionally, a quick-access config snapshot is saved in-place:
- `/opt/AdGuardHome/AdGuardHome.yaml.<version>` - config before migration, no need to unpack backup

**Note:** AdGuard Home automatically migrates `AdGuardHome.yaml` on startup when `schema_version` changes.
The backup and snapshot are taken before the new binary starts, so they always contain the pre-migration config.

### Rollback

Full rollback to previous version:

```bash
systemctl stop AdGuardHome
cp /opt/AdGuardHome.backup.v0.107.71/AdGuardHome /opt/AdGuardHome/AdGuardHome
cp /opt/AdGuardHome.backup.v0.107.71/AdGuardHome.yaml /opt/AdGuardHome/AdGuardHome.yaml
systemctl start AdGuardHome
```

Or rollback only binary (keep current config/data):

```bash
systemctl stop AdGuardHome
cp /opt/AdGuardHome.backup.v0.107.71/AdGuardHome /opt/AdGuardHome/AdGuardHome
systemctl start AdGuardHome
```

### List backups

```bash
ls -la /opt/ | grep AdGuardHome.backup
```

### Backup retention

Only the last 3 backups are kept. Older backups are automatically removed during update.

## Password Hash Generation

Generate bcrypt hash for admin password:

```bash
# Using htpasswd
htpasswd -B -n -b "" 'yourpassword' | cut -d: -f2

# Using Python
python3 -c "import bcrypt; print(bcrypt.hashpw(b'yourpassword', bcrypt.gensalt()).decode())"
```

## Examples

### Basic usage with WireGuard

```yaml
wireguard_address: "10.0.0.1/24"
wireguard_dns: "10.0.0.1"

adguardhome_admin_user: "admin"
adguardhome_admin_pass_hash: "{{ vault_adguardhome_pass_hash }}"
adguardhome_language: "en"
```

### Custom filters

```yaml
adguardhome_filters:
  - enabled: true
    url: "https://adguardteam.github.io/HostlistsRegistry/assets/filter_1.txt"
    name: "AdGuard DNS filter"
    id: 1
  - enabled: true
    url: "https://big.oisd.nl/"
    name: "OISD Big List"
    id: 2
```

### Whitelist rules

```yaml
adguardhome_user_rules:
  - '## Messengers'
  - '@@||telegram.org^$important'
  - '@@||*.telegram.org^$important'
  - '@@||t.me^$important'
```

## Service Management

```bash
# Check status
systemctl status AdGuardHome

# View logs
journalctl -u AdGuardHome -f

# Restart
systemctl restart AdGuardHome

# Check version
/opt/AdGuardHome/AdGuardHome --version
```

## Files

| File | Description |
|------|-------------|
| `/opt/AdGuardHome/` | Installation directory |
| `/opt/AdGuardHome/AdGuardHome` | Binary |
| `/opt/AdGuardHome/AdGuardHome.yaml` | Configuration |
| `/opt/AdGuardHome/AdGuardHome.yaml.<version>` | Config snapshot before update (pre-schema-migration) |
| `/opt/AdGuardHome/data/` | Databases and filters |
| `/opt/AdGuardHome.backup.<version>/` | Backup directory per version (last 3 kept): binary + yaml + data/ |
| `/etc/resolv.conf` | DNS configuration |
| `/etc/resolv.conf.orig` | Backup of original |
| `/etc/systemd/system/AdGuardHome.service` | Systemd service |

## Configuration Management

- Initial config is deployed via template on **fresh install only** (when binary was not present before)
- After first run, manage settings through Web UI
- Config will NOT be overwritten on subsequent runs or updates
- On update, only binary is replaced — config and data are preserved
- AdGuard Home auto-migrates `AdGuardHome.yaml` on startup if `schema_version` changes
- A pre-migration snapshot `AdGuardHome.yaml.<old_version>` is saved before each update

### Syncing filters, user rules and settings

With `adguardhome_sync_filtering: true` the two lists, and with
`adguardhome_sync_settings: true` the rate limit, client IP anonymization and query log
settings, stay managed by the inventory on existing installs too:

1. The role reads `AdGuardHome.yaml` and compares `filters` (`enabled`, `url`, `name`, `id`)
   and `user_rules` with `adguardhome_filters` / `adguardhome_user_rules` (plus `dns.ratelimit*`,
   `dns.anonymize_client_ip`, `querylog.enabled` / `file_enabled` with the settings sync).
2. No difference — nothing happens, the service is not touched.
3. Difference — the drift is printed, AdGuard Home is stopped, only these two keys are
   replaced, other keys untouched (the previous file is kept as a timestamped backup) and the
   service is started again. DNS is unavailable for a second or two.

The file is edited only while the service is stopped: AdGuard Home writes its in-memory
config back on changes and would overwrite an edit made while it runs. Lists changed in
the Web UI are reverted on the next run — keep them in the inventory.

## Troubleshooting

### Port 53 already in use

```bash
ss -tulpn | grep :53
```

Usually it's systemd-resolved. The role disables it by default.

### DNS not working after reboot

```bash
cat /etc/resolv.conf
```

Should contain fallback DNS servers.

### Cannot access web UI

1. Check service: `systemctl status AdGuardHome`
2. Check port: `ss -tlnp | grep 3000`
3. Connect via VPN first, then access `http://10.10.0.1:3000`

### Check binary version after update

```bash
/opt/AdGuardHome/AdGuardHome --version
```

## License

MIT
