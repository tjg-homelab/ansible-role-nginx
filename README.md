# Ansible Role: nginx

[![CI](https://github.com/tjg-homelab/ansible-role-nginx/actions/workflows/ci.yml/badge.svg)](https://github.com/tjg-homelab/ansible-role-nginx/actions/workflows/ci.yml)

Data-driven nginx for the reverse-proxy edge. Declare your vhosts as a list —
each one a **proxy**, **static** site, or **redirect** — and the role renders
the config: TLS termination, websocket upgrades, proxy headers, a catch-all
HTTP→HTTPS redirect, and an optional `stub_status` endpoint.

The design goal is that a vhost is *intent-level data*, not embedded nginx
syntax:

```yaml
nginx_vhosts:
  - name: sonarr.example.com
    mode: proxy
    upstream: http://192.168.3.19:8989
  - name: ha.example.com
    mode: proxy
    upstream: https://ha-backend.example.com
    websocket: true
```

This role does **not** obtain certificates — pair it with your cert tooling
(certbot, or a distribution role) and point `nginx_tls_certificate`/`_key`
at the result.

## Requirements

- Debian 12/13 or Ubuntu 22.04/24.04
- TLS certificate and key already present on the host (see above)

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `nginx_vhosts` | `[]` | List of vhosts to manage (schema below) |
| `nginx_tls_certificate` | `""` | Default cert (fullchain) for vhosts without their own |
| `nginx_tls_certificate_key` | `""` | Default private key |
| `nginx_ssl_protocols` | `TLSv1.2 TLSv1.3` | TLS protocols for all TLS vhosts |
| `nginx_ssl_ciphers` | `""` | Cipher string; empty = distro/openssl defaults |
| `nginx_http_redirect` | `true` | Catch-all port-80 server 301s everything to HTTPS |
| `nginx_acme_challenge` | `true` | Serve `/.well-known/acme-challenge/` from a webroot on :80 (for `certbot --webroot`) instead of redirecting it; only applies when `nginx_http_redirect` is true |
| `nginx_acme_webroot` | `/var/www/html` | Webroot the ACME challenge is served from |
| `nginx_remove_default_site` | `true` | Remove the distro default site |
| `nginx_status_enabled` | `false` | Serve `stub_status` on its own port |
| `nginx_status_port` | `8083` | Port for the status endpoint |
| `nginx_status_allow` | `[127.0.0.1]` | Sources allowed to read the status page |
| `nginx_proxy_set_headers` | X-Real-IP, X-Forwarded-* | Headers set on every proxied request |

### `nginx_vhosts` entry schema

```yaml
nginx_vhosts:
  - name: app.example.com        # required — primary server_name (also the filename)
    state: present               # optional — absent removes the vhost file
    mode: proxy                  # proxy (default) | static | redirect
    aliases:                     # optional — extra server_name entries
      - app-alias.example.com
    tls: true                    # optional — false = plain port-80 vhost
    tls_certificate: /path.pem   # optional — override the role-level default
    tls_certificate_key: /k.pem  # optional

    # mode: proxy
    upstream: http://127.0.0.1:8080   # proxy_pass target (verbatim; trailing
                                      # slash rewrites the path, none preserves
                                      # the raw/encoded URI)
    websocket: true              # optional — pass websocket upgrades through
    preserve_host: true          # optional — send the client Host upstream
    proxy_read_timeout: 600      # optional — seconds; also sets send timeout

    # mode: static
    root: /var/www/app           # docroot
    index: index.html index.htm  # optional

    # mode: redirect
    redirect_to: https://www.example.com   # target, no trailing slash
    redirect_code: 301           # optional
    redirect_preserve_path: true # optional — append the request path

    # any mode — extra location blocks
    locations:
      - path: /webhook           # nginx location match (modifiers allowed)
        upstream: http://127.0.0.1:1880/webhook   # proxied location…
        websocket: false         #   (same options as a proxy vhost)
      - path: "= /wiki/index.php"
        return: "302 https://wiki.example.com/"   # …or a return…
      - path: /assets/
        root: /var/www/assets                     # …or a docroot

    # last-resort escape hatch — raw lines inside the server block
    extra_config: |
      client_max_body_size 512m;
```

Notes:

- Per-vhost HTTP→HTTPS redirects are unnecessary — the role's single
  catch-all port-80 server handles every hostname. A cross-domain redirect
  takes two hops (HTTP→HTTPS on the same host, then the vhost's redirect),
  which is correct and keeps the data simple.
- HTTPS upstreams automatically get `proxy_ssl_server_name on` (SNI).
  Upstream certs are not verified, matching nginx's default.
- Websocket support uses a shared `map $http_upgrade $connection_upgrade`
  installed by the role.

## Example Playbook

```yaml
- hosts: edge
  roles:
    - role: nginx
      vars:
        nginx_tls_certificate: /etc/ssl/certs/wildcard.example.com.pem
        nginx_tls_certificate_key: /etc/ssl/keys/wildcard.example.com.key
        nginx_vhosts:
          - name: grafana.example.com
            mode: proxy
            upstream: http://127.0.0.1:3000
          - name: www.example.com
            mode: static
            root: /var/www/html/example.com
          - name: example.com
            mode: redirect
            redirect_to: https://www.example.com
```

## License

MIT

## Author

rnissen — part of the [tjg-homelab](https://github.com/tjg-homelab) role collection.
