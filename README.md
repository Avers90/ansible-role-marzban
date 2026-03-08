# ansible-role-marzban

Installs and configures [Marzban](https://github.com/Gozargah/Marzban) VPN panel (Xray-core) with VLESS TLS inbound and nginx fallback site.

Installed via `git clone` + Python virtualenv + systemd. No Docker required.

## Architecture

```
Client → :443 (VLESS TCP TLS) → Xray-core (systemd)
                                       ↓ fallback (non-VPN traffic)
                               127.0.0.1:8080 → nginx
                                                    ├── /dashboard/, /api/, /statics/, /sub/ → Marzban:8000
                                                    └── / → /var/www/html (static site)
HTTP :80 → nginx → 301 https://$host$request_uri
```

- Xray terminates TLS on port 443 and handles VLESS protocol
- Unrecognized HTTPS traffic is forwarded to `127.0.0.1:8080` (nginx)
- nginx proxies `/dashboard/`, `/api/`, `/statics/`, `/sub/` to Marzban on `127.0.0.1:8000`
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
| `marzban_data_dir` | `/var/lib/marzban` | Data directory (SQLite db, xray config, xray binary) |
| `marzban_cert_dir` | `/var/lib/marzban/certs` | TLS certificate directory |
| `marzban_venv_dir` | `{{ marzban_install_dir }}/venv` | Python virtualenv directory |
| `marzban_git_version` | `"v0.8.4"` | Git tag to install/update to |
| `marzban_xray_version` | `"v26.2.6"` | Xray-core version (pinned, update deliberately) |
| `marzban_xray_dir` | `{{ marzban_data_dir }}/xray-core` | Xray binary + assets directory |

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
| `marzban_xray_tls_min_version` | `"1.2"` | Minimum TLS version for VLESS inbound |
| `marzban_xray_cipher_suites` | `"TLS_ECDHE_..."` | TLS cipher suites (empty string = Xray defaults) |
| `marzban_xray_alpn` | `["http/1.1"]` | ALPN — do NOT add `h2`, breaks fallback to nginx |
| `marzban_xray_sniffing_enabled` | `true` | Enable Xray traffic sniffing |
| `marzban_xray_fallback_xver` | `0` | PROXY Protocol version sent to fallback (0 = disabled) |
| `marzban_xray_block_private_ip` | `true` | Block outbound to private/reserved IP ranges |
| `marzban_xray_captive_portal_fix` | `true` | Route connectivity check domains directly (fixes Hiddify timeout on connect) |
| `marzban_xray_extra_inbounds` | `[]` | Additional Xray inbounds appended after main VLESS TLS inbound |

### Nginx

| Variable | Default | Description |
|---|---|---|
| `marzban_nginx_config_deploy` | `true` | Deploy nginx vhost on `127.0.0.1:8080` |
| `marzban_nginx_fallback_port` | `8080` | Port nginx listens on for Xray fallback traffic |
| `marzban_site_root` | `"/var/www/html"` | Document root for static site |
| `marzban_site_access_log` | `"/var/log/nginx/marzban-access.log"` | Access log path |
| `marzban_site_error_log` | `"/var/log/nginx/marzban-error.log"` | Error log path |
| `marzban_site_security_headers` | `true` | Add `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy` headers |
| `marzban_subscription_path` | `"sub"` | Subscription URL path (proxied to Marzban) |

### Service

| Variable | Default | Description |
|---|---|---|
| `marzban_service_name` | `marzban` | systemd service name |
| `marzban_service_enabled` | `true` | Enable service on boot |
| `marzban_service_state` | `started` | Service state |

### Host settings (Marzban API)

Configured automatically via Marzban API after install. Controls what appears in Host Settings UI and gets embedded in client subscription links.

| Variable | Default | Description |
|---|---|---|
| `marzban_configure_hosts` | `true` | Configure host settings via API |
| `marzban_host_remark` | `"🚀 Marz ({USERNAME}) [...]"` | Display name in client apps |
| `marzban_host_security` | `"inbound_default"` | Security layer (`inbound_default\|tls\|none`) |
| `marzban_host_alpn` | `"http/1.1"` | ALPN in subscription links |
| `marzban_host_fingerprint` | `"chrome"` | uTLS fingerprint impersonation |
| `marzban_host_sni` | `" "` | SNI in subscription links — single space hides real domain from URI |
| `marzban_user_flow` | `""` | VLESS flow — keep empty (xtls-rprx-vision incompatible with h2mux/Hiddify) |

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

### Adding extra Xray inbounds

Use `marzban_xray_extra_inbounds` to append additional inbounds after the main VLESS TLS inbound.
Each entry is a plain YAML dict serialised to JSON:

```yaml
marzban_xray_extra_inbounds:
  - tag: "vless-ws"
    listen: "127.0.0.1"
    port: 8443
    protocol: "vless"
    settings:
      clients: []
      decryption: "none"
    streamSettings:
      network: "ws"
      wsSettings:
        path: "/vless-ws"
    sniffing:
      enabled: true
      destOverride: ["http", "tls", "quic"]
```

Certificate paths for extra inbounds (if TLS is terminated by Xray):

```
{{ marzban_cert_dir }}/{{ marzban_domain }}-fullchain.cer
{{ marzban_cert_dir }}/{{ marzban_domain }}.key
```

Use `ansible-vault` to protect `marzban_admin_password`.

## Admin account creation

The role uses `marzban-cli admin import-from-env` for non-interactive admin creation:

1. Temporarily injects `SUDO_USERNAME` / `SUDO_PASSWORD` into `.env`
2. Runs `marzban-cli admin import-from-env --yes`
3. Removes `SUDO_USERNAME` / `SUDO_PASSWORD` from `.env` immediately after

Idempotent — skipped if the username already exists in `marzban-cli admin list`.

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

# Update Marzban: bump marzban_git_version, re-run playbook
# Update Xray: bump marzban_xray_version, re-run playbook
```

## Firewall

This role does not configure firewall rules. Required open ports:

```yaml
firewall_ports_tcp:
  - 80   # acme.sh webroot challenge (Let's Encrypt HTTP-01)
  - 443  # Xray VLESS TLS
firewall_ddos_excluded_ports:
  - 443  # VLESS clients reconnect frequently
```

## Collections Required

None. This role uses only `ansible.builtin` modules.
