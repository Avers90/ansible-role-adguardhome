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
| `adguardhome_version` | `v0.107.71` | Version to install (always exact) |
| `adguardhome_update` | `false` | Force update if installed |
| `adguardhome_install_dir` | `/opt/AdGuardHome` | Installation directory |
| `adguardhome_disable_resolved` | `true` | Disable systemd-resolved |
| `adguardhome_configure_resolv` | `true` | Configure /etc/resolv.conf |
| `adguardhome_fallback_dns` | `[1.1.1.1, 8.8.8.8]` | Fallback DNS servers |
| `adguardhome_deploy_config` | `true` | Deploy initial config |
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
| `adguardhome_ratelimit` | `30` | Rate limit (requests/sec) |
| `adguardhome_filters` | AdGuard, AdAway | DNS filters |
| `adguardhome_user_rules` | `[]` | Custom whitelist rules |

## Version Management

**Important:** Always specify exact version in `defaults/main.yml`.

The template `AdGuardHome.yaml.j2` is tied to specific `schema_version`.
When updating AdGuard Home, you may need to update the template as well.

### Current version

```yaml
adguardhome_version: "v0.107.71"  # schema_version: 32
```

### Updating to new version

1. Check [AdGuard Home releases](https://github.com/AdguardTeam/AdGuardHome/releases)
2. Review changelog for config changes
3. Update `adguardhome_version` in `defaults/main.yml`
4. Update `templates/AdGuardHome.yaml.j2` if schema changed
5. Run update

### Version history

| Role version | AdGuard Home | schema_version |
|--------------|--------------|----------------|
| 1.0.0 | v0.107.71 | 32 |

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
adguardhome_language: "ru"
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
  # Messengers
  - "@@||telegram.org^$important"
  - "@@||*.telegram.org^$important"
  - "@@||t.me^$important"
```

## Backup and Restore

### Backup location

When updating, a full backup is created:

```
/opt/AdGuardHome.backup.<version>.<timestamp>/
```

Example:

```
/opt/AdGuardHome.backup.v0.107.71.20240115T143022/
```

### What's included in backup

- `AdGuardHome.yaml` - configuration
- `data/` - databases, statistics, query logs, downloaded filters

### List backups

```bash
ls -la /opt/ | grep AdGuardHome.backup
```

### Manual rollback

```bash
systemctl stop AdGuardHome
rm -rf /opt/AdGuardHome
mv /opt/AdGuardHome.backup.v0.107.71.20240115T143022 /opt/AdGuardHome
systemctl start AdGuardHome
```

### Clean old backups

```bash
rm -rf /opt/AdGuardHome.backup.*
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

## Files Modified

| File | Description |
|------|-------------|
| `/opt/AdGuardHome/` | Installation directory |
| `/opt/AdGuardHome/AdGuardHome.yaml` | Configuration |
| `/opt/AdGuardHome/data/` | Databases and filters |
| `/etc/resolv.conf` | DNS configuration |
| `/etc/resolv.conf.orig` | Backup of original |
| `/etc/systemd/system/AdGuardHome.service` | Systemd service |

## Configuration Management

- Initial config is deployed via template
- After first run, manage settings through Web UI
- Config will NOT be overwritten on subsequent runs
- On update, config and data are preserved from backup

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

## License

MIT
