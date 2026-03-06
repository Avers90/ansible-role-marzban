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

| Variable | Default | Description |
|---|---|---|
| `marzban_install_dir` | `/opt/marzban` | Installation directory |
| `marzban_data_dir` | `/var/lib/marzban` | Data directory (db, certs, xray config) |
| `marzban_cert_dir` | `/var/lib/marzban/certs` | TLS certificate directory |
| `marzban_domain` | `""` | **REQUIRED.** Domain name for TLS and subscription URL |
| `marzban_ssl_email` | `admin@example.com` | Email for acme.sh Let's Encrypt notifications |
| `marzban_panel_port` | `8000` | Marzban panel internal port |
| `marzban_uvicorn_host` | `0.0.0.0` | Uvicorn listen address |
| `marzban_admin_username` | `""` | **REQUIRED.** Admin username |
| `marzban_admin_password` | `""` | **REQUIRED.** Admin password |
| `marzban_xray_config_deploy` | `true` | Deploy custom `xray_config.json` with VLESS TLS + fallback |
| `marzban_nginx_config_deploy` | `true` | Deploy nginx fallback vhost on port 8080 |
| `marzban_xray_port` | `443` | Xray VLESS TLS listen port |
| `marzban_nginx_fallback_port` | `8080` | Nginx fallback port (Xray → nginx) |
| `marzban_service_name` | `marzban` | Docker compose project name |
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

## Credentials

Admin credentials are saved to `{{ marzban_install_dir }}/credentials.txt` (mode `0600`).  
**Delete this file after saving credentials elsewhere.**

## Firewall

This role does not configure firewall rules. Add to `ansible-role-firewall` variables:

```yaml
firewall_ports_tcp:
  - 80   # acme.sh webroot challenge + HTTP redirect
  - 443  # Xray VLESS TLS
```

## Collections Required

- `community.docker` — `docker_compose_v2`

```bash
ansible-galaxy collection install community.docker
```
