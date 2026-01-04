# ansible-role-wgdashboard

Install and configure WGDashboard - web UI for WireGuard.

## Requirements

- Debian/Ubuntu
- WireGuard installed (use ansible-role-wireguard)

## Supported Distributions

- Debian (bullseye, bookworm)
- Ubuntu (focal, jammy, noble)

Other distributions will fail with an error message.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `wgdashboard_version` | `latest` | Version tag from GitHub |
| `wgdashboard_install_dir` | `/opt/wgdashboard` | Installation directory |
| `wgdashboard_wg_interface` | `{{ wireguard_interface }}` | WireGuard interface (for systemd) |

## Default Credentials

After installation, login with:

- **Username:** admin
- **Password:** admin

**⚠️ CHANGE THE PASSWORD IMMEDIATELY AFTER FIRST LOGIN!**

## Configuration

All settings are managed through the WGDashboard GUI:

- Host/port
- Theme
- Language
- Peer defaults (DNS, MTU, etc.)
- Autostart interfaces

Ansible only installs and updates the application code, it does not modify `wg-dashboard.ini`.

## Access Methods

By default, WGDashboard listens on `127.0.0.1:10086`.

### SSH Tunnel (recommended)

```bash
ssh -L 10086:127.0.0.1:10086 user@server
```

Then open: <http://localhost:10086>

### Nginx Reverse Proxy (production)

Use `ansible-role-nginx` to configure reverse proxy with SSL.

## Examples

### Specific version

```yaml
wgdashboard_version: "v4.3.1"
```

## Files Created

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
