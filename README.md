# ansible-role-wgdashboard

Install and configure [WGDashboard](https://github.com/WGDashboard/WGDashboard) - web UI for WireGuard.

## Requirements

- Debian/Ubuntu
- Python 3.12+ on the target host (required since WGDashboard v4.3.3)
- WireGuard installed (use [ansible-role-wireguard](https://github.com/Avers90/ansible-role-wireguard))
- Collections: `ansible.utils`, `community.general`

Other distributions will fail with an error message.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `wgdashboard_version` | `v4.3.3` | Version tag from GitHub |
| `wgdashboard_install_dir` | `/opt/WGDashboard` | Installation directory |
| `wgdashboard_wg_interface` | `{{ wireguard_interface }}` | WireGuard interface |
| `wgdashboard_admin_user` | `admin` | Admin username |
| `wgdashboard_admin_pass_hash` | `""` | Bcrypt password hash (empty = default 'admin') |
| `wgdashboard_app_ip` | `127.0.0.1` | Listen IP address |
| `wgdashboard_app_port` | `10086` | Listen port |
| `wgdashboard_peer_global_dns` | from `wireguard_address` | Default DNS for peers |
| `wgdashboard_peer_mtu` | `1420` | Default MTU for peers |
| `wgdashboard_peer_keep_alive` | `21` | Default keep-alive for peers |
| `wgdashboard_peer_remote_endpoint` | `{{ ansible_host }}` | Public IP/hostname WGDashboard puts into peer `Endpoint = ...:port`. Without this WGDashboard resolves via `gethostbyname()` → `127.0.1.1` from default Debian `/etc/hosts` |
| `wgdashboard_peer_endpoint_allowed_ip` | `0.0.0.0/0,::/0` | Default `AllowedIPs` for new peers (full-tunnel). Override to e.g. `10.0.0.0/24` for split-tunnel |

## Password Hash Generation

Generate bcrypt hash for admin password:

```bash
# Using htpasswd
htpasswd -B -n -b "" 'yourpassword' | cut -d: -f2

# Using Python
python3 -c "import bcrypt; print(bcrypt.hashpw(b'yourpassword', bcrypt.gensalt()).decode())"
```

Store in vault:

```yaml
# inventory/(group|host)_vars/.../vault.yml
vault_wgdashboard_pass_hash: "$2b$12$..."
```

Use in variables:

```yaml
wgdashboard_admin_pass_hash: "{{ vault_wgdashboard_pass_hash }}"
```

## Examples

### Basic usage

```yaml
wgdashboard_admin_user: "aver"
wgdashboard_admin_pass_hash: "{{ vault_wgdashboard_pass_hash }}"
wgdashboard_peer_global_dns: "10.110.0.1"
wgdashboard_peer_mtu: 1380          # override per host if PMTU issues observed
wgdashboard_peer_keep_alive: 15
# wgdashboard_peer_remote_endpoint: "{{ ansible_host }}"   # default
# wgdashboard_peer_endpoint_allowed_ip: "0.0.0.0/0,::/0"   # default (full-tunnel)
```

### Split-tunnel example

```yaml
# Route only company subnet through VPN
wgdashboard_peer_endpoint_allowed_ip: "10.50.0.0/16"
```

### Custom endpoint hostname

```yaml
# Use DNS instead of IP (requires A record for vpn.example.com)
wgdashboard_peer_remote_endpoint: "vpn.example.com"
```

## Version Management

**Important:** Always specify exact version in `defaults/main.yml`.

### Current version

```yaml
wgdashboard_version: "v4.3.3"
```

### Updating to new version

1. Check [WGDashboard releases](https://github.com/WGDashboard/WGDashboard/releases)
2. Update `wgdashboard_version` in `defaults/main.yml`
3. Run playbook — it will detect version mismatch and update

> **Note (v4.3.3+):** Python 3.12+ is required. If the host is running Python 3.10/3.11,
> upgrade Python manually first, then remove the old venv before running the playbook:
> ```bash
> rm -rf /opt/WGDashboard/src/venv
> ```

### What happens during update

1. Stop WGDashboard service
2. Create backup of user data (see below)
3. `git fetch --tags`
4. `git checkout <version>`
5. `./wgd.sh install`
6. Fix file permissions
7. Restart service

### Backup before update

Before every update, the role automatically creates a versioned backup of all user data:

```
/opt/WGDashboard.backup.<old-version>/
├── db/                    # all SQLite databases
│   ├── wgdashboard.db     # peers, traffic stats, API keys, share links
│   └── wgdashboard_job.db # scheduled jobs
├── wg-dashboard.ini       # dashboard config (credentials, settings)
└── ssl-tls.ini            # SSL config (if present)
```

The last **3 backups** are retained; older ones are removed automatically.

Note: `/etc/wireguard/*.conf` files are not backed up — they live outside the repository and are never touched by `git checkout`.

### What's preserved during update

- `wg-dashboard.ini` — configuration
- `db/` — database with all peers and traffic history
- Users and passwords
- Peer settings

## Default Credentials

After installation, login with:

- **Username:** wgdashboard_admin_user
- **Password:** wgdashboard_admin_pass_hash

**⚠️ CHANGE THE PASSWORD IMMEDIATELY AFTER FIRST LOGIN IF YOU ARE admin!**

## Configuration

Settings are split between Ansible-managed (idempotent across runs) and GUI-managed:

**Ansible-managed** (rewritten on every play, override via host_vars):

- Account (admin user/password — only set on fresh install)
- Server (`app_ip`, `app_port`)
- **Peers (`peer_global_dns`, `peer_mtu`, `peer_keep_alive`, `remote_endpoint`,
  `peer_endpoint_allowed_ip`)** — синхронизируются при каждом прогоне
- WireGuardConfiguration (`autostart`)
- Other (`welcome_session`)

**GUI-managed** (changes preserved in `wg-dashboard.ini` between Ansible runs):

- Theme
- Language
- API keys
- TOTP setup details

If you change a Peer-related variable in `host_vars` and re-run the playbook —
new value is applied to `wg-dashboard.ini` immediately, but **existing peers**
keep their per-peer settings (MTU, AllowedIPs, etc.) — only **new** peers pick
up the new defaults. To migrate existing peers, edit them via GUI or call
WGDashboard API.

## Access Methods

By default, WGDashboard listens on `127.0.0.1:10086`.

### SSH Tunnel (recommended)

```bash
ssh user@server -L 10086:127.0.0.1:10086 -N
```

Then open: <http://localhost:10086>

## Security

This role automatically fixes file permissions in `/etc/wireguard` after WGDashboard installation:

| File | Permissions |
|------|-------------|
| `*.conf` | `0600` |
| `*_privatekey` | `0600` |
| `*_publickey` | `0644` |

WGDashboard's install script sets `755` on these files, which is insecure.

## Files

| File | Description |
|------|-------------|
| `/opt/WGDashboard/` | Application directory |
| `/opt/WGDashboard/src/wg-dashboard.ini` | Configuration (managed via GUI) |
| `/opt/WGDashboard/src/db/wgdashboard.db` | SQLite database (peers, traffic, API keys) |
| `/opt/WGDashboard/src/db/wgdashboard_job.db` | Scheduled jobs database |
| `/opt/WGDashboard.backup.<version>/` | Versioned backups (last 3 kept) |
| `/etc/systemd/system/wg-dashboard.service` | Systemd service |

## Service Management

```bash
# Check status
systemctl status wg-dashboard

# View logs
journalctl -u wg-dashboard -f

# Restart
systemctl restart wg-dashboard

# Using wgd.sh
cd /opt/WGDashboard/src
./wgd.sh start
./wgd.sh stop
./wgd.sh restart
./wgd.sh update
```

## License

MIT
