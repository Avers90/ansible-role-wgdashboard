# ansible-role-wgdashboard

Install and configure [WGDashboard](https://github.com/WGDashboard/WGDashboard) - web UI for WireGuard.

## Requirements

- Debian/Ubuntu
- WireGuard installed (use [ansible-role-wireguard](https://github.com/Avers90/ansible-role-wireguard))
- Collections: `ansible.utils`, `community.general`

Other distributions will fail with an error message.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `wgdashboard_version` | `v4.3.1` | Version tag from GitHub |
| `wgdashboard_install_dir` | `/opt/wgdashboard` | Installation directory |
| `wgdashboard_wg_interface` | `{{ wireguard_interface }}` | WireGuard interface |
| `wgdashboard_admin_user` | `admin` | Admin username |
| `wgdashboard_admin_pass_hash` | `""` | Bcrypt password hash (empty = default 'admin') |
| `wgdashboard_app_ip` | `127.0.0.1` | Listen IP address |
| `wgdashboard_app_port` | `10086` | Listen port |
| `wgdashboard_peer_global_dns` | from `wireguard_address` | Default DNS for peers |
| `wgdashboard_peer_mtu` | `1420` | Default MTU for peers |
| `wgdashboard_peer_keep_alive` | `21` | Default keep-alive for peers |

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
wgdashboard_peer_mtu: 1380
wgdashboard_peer_keep_alive: 15
```

## Version Management

**Important:** Always specify exact version in `defaults/main.yml`.

### Current version

```yaml
wgdashboard_version: "v4.3.1"
```

### Updating to new version

1. Check [WGDashboard releases](https://github.com/WGDashboard/WGDashboard/releases)
2. Update `wgdashboard_version` in `defaults/main.yml`
3. Run playbook — it will detect version mismatch and update

### What happens during update

1. Stop WGDashboard service
2. `git fetch --tags`
3. `git checkout <version>`
4. `./wgd.sh install`
5. Fix file permissions
6. Restart service

### What's preserved during update

- `wg-dashboard.ini` — configuration
- `db/` — database
- Users and passwords
- Peer settings

## Default Credentials

After installation, login with:

- **Username:** wgdashboard_admin_user
- **Password:** wgdashboard_admin_pass_hash

**⚠️ CHANGE THE PASSWORD IMMEDIATELY AFTER FIRST LOGIN IF YOU ARE admin!**

## Configuration

After first run, all settings are managed through the WGDashboard GUI:

- Host/port
- Theme
- Language
- Peer defaults (DNS, MTU, etc.)
- Autostart interfaces

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
| `/opt/wgdashboard/` | Application directory |
| `/opt/wgdashboard/src/wg-dashboard.ini` | Configuration (managed via GUI) |
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
cd /opt/wgdashboard/src
./wgd.sh start
./wgd.sh stop
./wgd.sh restart
./wgd.sh update
```

## License

MIT
