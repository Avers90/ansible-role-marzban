# ansible-role-marzban

Installs and configures [Marzban](https://github.com/Gozargah/Marzban) VPN panel (Xray-core) with VLESS TLS inbound and nginx decoy fallback.

## Architecture

```
Client → :443 (VLESS TLS) → Xray-core
                                 ↓ fallback (non-VPN traffic)
                         127.0.0.1:8080 → nginx → /var/www/html
```

- Xray terminates TLS on port 443
- Unrecognized traffic is forwarded to `127.0.0.1:8080` where nginx serves a static decoy site
- The panel dashboard is accessible at `https://<domain>/dashboard/`
- Certificates are issued via `acme.sh` using nginx webroot (`/var/www/html`)

## Requirements

- Docker CE installed (`ansible-role-docker`)
- nginx installed and running (`ansible-role-nginx`) — required for acme.sh webroot challenge

**Recommended execution order:**
```yaml
roles:
  - ansible-role-docker
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

### Directories

| Variable | Default | Description |
|---|---|---|
| `marzban_install_dir` | `/opt/marzban` | Installation directory (docker-compose.yml, .env) |
| `marzban_data_dir` | `/var/lib/marzban` | Data directory (SQLite db, xray config) |
| `marzban_cert_dir` | `/var/lib/marzban/certs` | TLS certificate directory |

### Panel

| Variable | Default | Description |
|---|---|---|
| `marzban_panel_port` | `8000` | Marzban panel internal port |
| `marzban_uvicorn_host` | `"0.0.0.0"` | Uvicorn listen address |
| `marzban_dashboard_path` | `"/dashboard/"` | Dashboard URL path |
| `marzban_ssl_email` | `"admin@example.com"` | Email for Let's Encrypt notifications |
| `marzban_container_name` | `"marzban-marzban-1"` | Docker container name (project-service-1 format) |

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

### Nginx decoy

| Variable | Default | Description |
|---|---|---|
| `marzban_nginx_config_deploy` | `true` | Deploy nginx fallback vhost on `127.0.0.1:8080` |
| `marzban_nginx_fallback_port` | `8080` | Port nginx listens on for Xray fallback traffic |
| `marzban_decoy_root` | `"/var/www/html"` | Document root for decoy site |
| `marzban_decoy_access_log` | `"/var/log/nginx/marzban-decoy-access.log"` | Access log path |
| `marzban_decoy_error_log` | `"/var/log/nginx/marzban-decoy-error.log"` | Error log path |
| `marzban_decoy_security_headers` | `true` | Add `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy` headers |

### Service

| Variable | Default | Description |
|---|---|---|
| `marzban_service_name` | `marzban` | Docker Compose project name |
| `marzban_service_enabled` | `true` | Enable service on boot |
| `marzban_service_state` | `started` | Service state |

## Example Playbook

```yaml
- hosts: vps
  roles:
    - role: ansible-role-docker
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
2. Runs `marzban-cli admin import-from-env --yes` inside the container
3. Removes `SUDO_USERNAME` / `SUDO_PASSWORD` from `.env` immediately after

This is idempotent — the task is skipped if the username already exists in `marzban-cli admin list`.

## Credentials

Admin credentials are saved to `{{ marzban_install_dir }}/credentials.txt` (mode `0600`).
**Delete this file after saving credentials elsewhere.**

```bash
ssh aver@<host> sudo cat /opt/marzban/credentials.txt
```

## Firewall

This role does not configure firewall rules. Add to `ansible-role-firewall` variables:

```yaml
firewall_ports_tcp:
  - 80   # acme.sh webroot challenge (Let's Encrypt HTTP-01)
  - 443  # Xray VLESS TLS
```

## Collections Required

- `community.docker` — `docker_compose_v2`

```bash
ansible-galaxy collection install community.docker
```
