# ansible-role-marzban

Installs and configures [Marzban](https://github.com/Gozargah/Marzban) VPN panel (Xray-core) with VLESS TLS inbound and nginx fallback site.

Installed via `git clone` + Python virtualenv + systemd. No Docker required.

## Architecture

```
Client → :443 (VLESS TLS) → Xray-core (systemd)
                                  ↓ fallback (non-VPN traffic)
                          127.0.0.1:8080 → nginx
                                               ├── /dashboard/, /api/ → Marzban:8000
                                               └── / → /var/www/html (static site)
```

- Xray terminates TLS on port 443 and handles VLESS protocol
- Unrecognized HTTPS traffic is forwarded to `127.0.0.1:8080` (nginx)
- nginx proxies `/dashboard/` and `/api/` to Marzban panel on `127.0.0.1:8000`
- nginx serves `/var/www/html` for all other traffic (looks like a real website)
- TLS certificates issued via `acme.sh` using nginx webroot (`/var/www/html`)
- Marzban listens on `127.0.0.1:8000` only — not directly exposed

## Requirements

- nginx installed and running (`ansible-role-nginx`) — required for acme.sh webroot challenge

**Recommended execution order:**
```yaml
roles:
  - ansible-role-nginx
  - ansible-role-marzban
```

## Variables

### Required

| Variable | Default | Description |
|---|---|---|
| `marzban_domain` | `""` | Domain name for TLS certificate and subscription URL |
| `marzban_admin_username` | `""` | Admin account username |
| `marzban_admin_password` | `""` | Admin account password (use ansible-vault) |

### Installation

| Variable | Default | Description |
|---|---|---|
| `marzban_install_dir` | `/opt/marzban` | Git clone directory |
| `marzban_data_dir` | `/var/lib/marzban` | Data directory (SQLite db, xray config) |
| `marzban_cert_dir` | `/var/lib/marzban/certs` | TLS certificate directory |
| `marzban_venv_dir` | `{{ marzban_install_dir }}/venv` | Python virtualenv directory |
| `marzban_git_version` | `"master"` | Git branch or tag to install |
| `marzban_xray_version` | `"latest"` | Xray version for Xray-install script |

### Panel

| Variable | Default | Description |
|---|---|---|
| `marzban_panel_port` | `8000` | Marzban panel internal port (listens on 127.0.0.1) |
| `marzban_dashboard_path` | `"/dashboard/"` | Dashboard URL path |
| `marzban_ssl_email` | `"admin@example.com"` | Email for Let's Encrypt notifications |

### Xray

| Variable | Default | Description |
|---|---|---|
| `marzban_xray_config_deploy` | `true` | Deploy `xray_config.json` with VLESS TLS + fallback |
| `marzban_xray_port` | `443` | Xray VLESS TLS listen port |
| `marzban_xray_listen` | `"0.0.0.0"` | Xray listen address |
| `marzban_xray_log_level` | `"warning"` | Xray log level (`debug\|info\|warning\|error\|none`) |
| `marzban_xray_alpn` | `["h2","http/1.1"]` | ALPN protocols in TLS handshake |
| `marzban_xray_sniffing_enabled` | `true` | Enable Xray traffic sniffing |
| `marzban_xray_fallback_xver` | `0` | PROXY Protocol version sent to fallback (0 = disabled) |
| `marzban_xray_block_private_ip` | `true` | Block outbound to private/reserved IP ranges |

### Nginx

| Variable | Default | Description |
|---|---|---|
| `marzban_nginx_config_deploy` | `true` | Deploy nginx vhost on `127.0.0.1:8080` |
| `marzban_nginx_fallback_port` | `8080` | Port nginx listens on for Xray fallback traffic |
| `marzban_site_root` | `"/var/www/html"` | Document root for static site |
| `marzban_site_access_log` | `"/var/log/nginx/marzban-access.log"` | Access log path |
| `marzban_site_error_log` | `"/var/log/nginx/marzban-error.log"` | Error log path |
| `marzban_site_security_headers` | `true` | Add `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy` headers |

### Service

| Variable | Default | Description |
|---|---|---|
| `marzban_service_name` | `marzban` | systemd service name |
| `marzban_service_enabled` | `true` | Enable service on boot |
| `marzban_service_state` | `started` | Service state |

## Example Playbook

```yaml
- hosts: vps
  roles:
    - role: ansible-role-nginx
    - role: ansible-role-marzban
      vars:
        marzban_domain: "vpn.example.com"
        marzban_ssl_email: "admin@example.com"
        marzban_admin_username: "admin"
        marzban_admin_password: "{{ vault_marzban_password }}"
```

Use `ansible-vault` to protect `marzban_admin_password`.

## Admin account creation

The role uses `marzban-cli admin import-from-env` for non-interactive admin creation:

1. Temporarily injects `SUDO_USERNAME` / `SUDO_PASSWORD` into `.env`
2. Runs `marzban-cli admin import-from-env --yes` (`python-decouple` reads `.env` directly)
3. Removes `SUDO_USERNAME` / `SUDO_PASSWORD` from `.env` immediately after

This is idempotent — skipped if the username already exists in `marzban-cli admin list`.

## Credentials

Admin credentials are saved to `{{ marzban_install_dir }}/credentials.txt` (mode `0600`).
**Delete this file after saving credentials elsewhere.**

```bash
ssh user@<host> sudo cat /opt/marzban/credentials.txt
```

## Useful commands

```bash
# Service management
systemctl status marzban
journalctl -u marzban -f

# Admin management
marzban-cli admin list
marzban-cli admin create --sudo

# Update (bump marzban_git_version, then re-run playbook)
# Role runs git pull + pip install + alembic upgrade head + service restart
```

## Firewall

This role does not configure firewall rules. Required open ports:

```yaml
firewall_ports_tcp:
  - 80   # acme.sh webroot challenge (Let's Encrypt HTTP-01)
  - 443  # Xray VLESS TLS
```

## Collections Required

None. This role uses only `ansible.builtin` modules.
