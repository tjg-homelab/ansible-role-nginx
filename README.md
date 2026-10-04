# Ansible Role: nginx

[![CI](https://github.com/tjg-homelab/ansible-role-nginx/actions/workflows/ci.yml/badge.svg)](https://github.com/tjg-homelab/ansible-role-nginx/actions/workflows/ci.yml)

Data-driven nginx for the reverse-proxy edge. Declare your vhosts as a list —
each one a **proxy**, **static** site, **redirect**, or **php** app (FastCGI to
php-fpm) — and the role renders the config: TLS termination, websocket upgrades,
proxy headers, a catch-all HTTP→HTTPS redirect, an optional `stub_status`
endpoint, a dotfile deny on every vhost, and opt-in security headers.

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
| `nginx_https_reject_unknown` | `true` | Catch-all :443 server that refuses the handshake for unknown SNI / bare-IP requests (nginx 1.19.4+) |
| `nginx_ssl_session_cache` | `shared:SSL:10m` | TLS session cache on every TLS vhost; empty = leave unset |
| `nginx_ssl_session_timeout` | `1d` | TLS session lifetime (with the cache above) |
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
| `nginx_proxy_redirect_https` | `true` | On TLS vhosts, rewrite a backend's absolute `http://` redirects to `https://` (ProxyPassReverse parity) |
| `nginx_proxy_client_max_body_size` | `""` | `client_max_body_size` for `mode: proxy` vhosts without their own; empty = inherit the http-level value |
| `nginx_default_security_headers` | `[]` | Response headers added (with `always`) to every vhost — e.g. `["Strict-Transport-Security max-age=15768000"]`. Empty by default; see **Security headers** below before enabling alongside a separate hardening role |

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
    security_headers: true       # optional — false excludes this vhost from
                                 # nginx_default_security_headers

    # mode: proxy
    upstream: http://127.0.0.1:8080   # proxy_pass target (verbatim; trailing
                                      # slash rewrites the path, none preserves
                                      # the raw/encoded URI)
    preserve_host: true          # optional — send the client Host upstream
    proxy_read_timeout: 600      # optional — seconds; also sets send timeout
    buffering: false             # optional — proxy_buffering off (streaming /
                                 # event-stream apps, e.g. Home Assistant)
    proxy_redirect_https: false  # optional — opt out of the http:// -> https://
                                 # Location rewrite (on by default for TLS vhosts)
    client_max_body_size: 50m    # optional, any mode — overrides
                                 # nginx_proxy_client_max_body_size
    websocket: true              # deprecated no-op — upgrades always pass through

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
        buffering: false         #   (same options as a proxy vhost)
      - path: "= /wiki/index.php"
        return: "302 https://wiki.example.com/"   # …or a return…
      - path: /assets/
        root: /var/www/assets                     # …or a docroot

    # last-resort escape hatch — raw lines inside the server block
    extra_config: |
      add_header X-Robots-Tag noindex;
```

Notes:

- Per-vhost HTTP→HTTPS redirects are unnecessary — the role's single
  catch-all port-80 server handles every hostname. A cross-domain redirect
  takes two hops (HTTP→HTTPS on the same host, then the vhost's redirect),
  which is correct and keeps the data simple.
- HTTPS upstreams automatically get `proxy_ssl_server_name on` (SNI).
  Upstream certs are not verified, matching nginx's default.
- Every proxied location speaks HTTP/1.1 upstream and passes websocket
  upgrades through, via a shared `map $http_upgrade $connection_upgrade`
  installed by the role (a plain request gets `Connection: close`, exactly as
  before). No per-vhost flag is needed.
- On TLS vhosts, redirects from the backend are rewritten the way Apache's
  `ProxyPassReverse` would: `proxy_redirect default` (Locations naming the
  upstream address) plus `proxy_redirect http:// https://` (a backend that
  sees the real Host but not the TLS in front of it). Without the second
  rule the browser is bounced to :80, and a redirected POST arrives as a
  bodiless GET.
- `X-Forwarded-Host` is sent alongside the other X-Forwarded-* headers.
- `nginx_proxy_client_max_body_size` sets a body limit for proxy vhosts only,
  so a hardening layer can keep a tiny global limit for static sites
  without returning 413 on ordinary API writes. A vhost's own
  `client_max_body_size` wins; so does one already set in `extra_config`.
- A catch-all `:443` server (`nginx_https_reject_unknown`) refuses the TLS
  handshake for any name no vhost claims, including requests to the bare IP,
  instead of answering with the first-loaded vhost's certificate and backend.

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

## Dotfile protection (AN-23)

Every vhost, in every mode, denies `/\.` except `/.well-known/`:

```nginx
location ~ /\.(?!well-known/) {
    deny all;
    return 404;
}
```

This is not configurable — a docroot serving `.env`, `.git/`, `.htaccess`, or
an accidentally-shipped `.gitignore` straight to the public internet is never
correct, so there is no vhost-level opt-out. `/.well-known/` is excluded so
other legitimate uses (e.g. `security.txt`) keep working; ACME's own HTTP-01
carve-out is unaffected either way, since that's a separate catch-all on
port 80 (`nginx_acme_challenge`), not this per-vhost TLS server block.

## Security headers

`nginx_default_security_headers` is **empty by default** — deliberately, not
an oversight. Read the comment on that variable in `defaults/main.yml` before
setting it: `add_header` inside a server block discards every header
inherited from a parent (e.g. http-level) context, so turning this on where
this role is *already* paired with a separate hardening role (like
`devsec.hardening.nginx_hardening`) will silently **replace**, not add to,
whatever that layer is providing.

Enable it when this role runs **without** a separate hardening layer:

```yaml
nginx_default_security_headers:
  - "Strict-Transport-Security max-age=15768000"
  - "X-Frame-Options SAMEORIGIN"
  - "X-Content-Type-Options nosniff"
```

Each entry is rendered as `add_header <entry> always;` — the `always`
parameter means it lands on error responses (404, 403, 502, ...) too, not
just the 2xx/3xx nginx applies `add_header` to by default. Set
`security_headers: false` on a specific vhost to exclude just that one.

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
