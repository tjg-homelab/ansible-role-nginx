# Ansible Role: nginx

[![CI](https://github.com/tjg-homelab/ansible-role-nginx/actions/workflows/ci.yml/badge.svg)](https://github.com/tjg-homelab/ansible-role-nginx/actions/workflows/ci.yml)

Data-driven nginx for the reverse-proxy edge. Declare your vhosts as a list —
each one a **proxy**, **static** site, **redirect**, or **php** app (FastCGI to
php-fpm) — and the role renders the config: TLS termination, websocket upgrades,
proxy headers, a catch-all HTTP→HTTPS redirect, and an optional `stub_status`
endpoint.

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
| `nginx_robots_enabled` | `true` | Serve `/robots.txt` on every vhost from one host-wide policy |
| `nginx_robots_content` | AI/scraper blocklist | The policy body (see below) |
| `nginx_robots_extra` | `""` | Rules appended into the trailing `User-agent: *` stanza |
| `nginx_php_fpm_pass` | `unix:/run/php/php8.4-fpm.sock` | Default FastCGI upstream for `mode: php` vhosts (override per vhost with `php_fpm_pass`) |
| `nginx_status_enabled` | `false` | Serve `stub_status` on its own port |
| `nginx_status_port` | `8083` | Port for the status endpoint |
| `nginx_status_allow` | `[127.0.0.1]` | Sources allowed to read the status page |
| `nginx_proxy_set_headers` | X-Real-IP, X-Forwarded-* | Headers set on every proxied request |

### `nginx_vhosts` entry schema

```yaml
nginx_vhosts:
  - name: app.example.com        # required — primary server_name (also the filename)
    state: present               # optional — absent removes the vhost file
    mode: proxy                  # proxy (default) | static | redirect | php
    aliases:                     # optional — extra server_name entries
      - app-alias.example.com
    tls: true                    # optional — false = plain port-80 vhost
    tls_certificate: /path.pem   # optional — override the role-level default
    tls_certificate_key: /k.pem  # optional
    robots: true                 # optional — false omits the shared robots.txt
                                 # include (for a vhost that serves its own, or
                                 # must pass /robots.txt to a backend)

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

    # mode: php   (FastCGI to php-fpm — e.g. phpBB, WordPress, plain PHP)
    root: /var/www/forum         # required — docroot
    index: index.php index.html  # optional (default: index.php index.html)
    php_fpm_pass: 127.0.0.1:9010 # optional — override nginx_php_fpm_pass (socket or host:port)
    php_front_controller: /app.php   # optional — unmatched URIs fall through to
                                     # this script (phpBB/Symfony routing); omit
                                     # for plain PHP (unmatched → 404)
    php_deny:                    # optional — location regexes returned as 403
      - /(config|cache|files|includes|store|vendor)

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

## robots.txt

The role serves one host-wide `/etc/nginx/robots.txt` on every vhost it
generates, via a `location = /robots.txt` snippet included in each server
block. `location =` is an exact match, so it wins over a `mode: php` front
controller or a `proxy_pass` at `location /` regardless of block order.

This exists because "no robots.txt" is not a neutral state. On a `mode: php`
vhost with a front controller the request falls through to the application and
returns HTML with `200`, which crawlers read as *no restrictions* — strictly
worse than a 404.

**The default policy blocks AI/LLM training and scraping agents only**
(GPTBot, ClaudeBot, CCBot, Google-Extended, PerplexityBot, Bytespider,
Amazonbot and friends) and applies `Crawl-delay: 10` to everything else.
Googlebot and Bingbot are deliberately *not* blocked — a role default that
deindexes the site from search is the wrong default. `Google-Extended` is
Google's AI-training opt-out token and has no effect on Search indexing or
ranking, which is why it can be blocked safely.

Add site-specific rules with `nginx_robots_extra` rather than restating the
agent list:

```yaml
nginx_robots_extra: |
  Disallow: /ucp.php
  Disallow: /search.php
```

Two behaviours worth knowing:

- **`mode: redirect` vhosts do not serve it.** A server-level `return 301` runs
  in the rewrite phase, before location matching, so `/robots.txt` is
  redirected to the canonical host and served there.
- **`mode: proxy` vhosts intercept it.** The backend never sees `/robots.txt`.
  Set `robots: false` on that vhost if the backend must serve its own.

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
