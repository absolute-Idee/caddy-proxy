# caddy-proxy

Caddy with the [naive fork of forwardproxy](https://github.com/klzgrad/forwardproxy), running in Docker. To everyone else the domain is a small static blog with a valid Let's Encrypt certificate. Requests with the right proxy credentials are tunneled as an HTTPS proxy.

```
Browser + FoxyProxy --TLS 443, CONNECT + Proxy-Authorization--> Caddy --> forward_proxy --> Internet
Probe / visitor     --TLS 443, GET or CONNECT without auth----> Caddy --> file_server (site/)
```

- One port (443/tcp), a real domain and a real certificate. The outer TLS handshake is your browser's own.
- `probe_resistance`: without valid credentials, proxy requests get the website's normal response, never a `407`. `hide_ip` and `hide_via` strip identifying headers.
- HTTP/3 is disabled and UDP 443 is not published.
- The admin API is off, and the `Server` header is removed.

## Layout

| Path | Purpose |
| --- | --- |
| `Dockerfile` | Builds Caddy with `forwardproxy@naive` via xcaddy |
| `docker-compose.yml` | Runs the container, ports 80 and 443/tcp, persistent cert volumes |
| `Caddyfile` | Proxy + static site config, values come from `.env` |
| `.env.example` | Template for domain, email, credentials, probe domain |
| `site/` | The fake website. Edit freely; it's mounted read-only. |

## Deploy

1. **DNS.** Create an A record for your domain pointing at the VPS IP. DNS only: no Cloudflare orange cloud or other CDN proxying.

2. **VPS.** Install Docker (with the compose plugin) and set up the firewall:

   ```sh
   curl -fsSL https://get.docker.com | sh
   ufw allow 22/tcp
   ufw allow 80/tcp
   ufw allow 443/tcp
   ufw deny 443/udp
   ufw enable
   ```

3. **Configure and start.**

   ```sh
   git clone <this repo> caddy-proxy && cd caddy-proxy
   cp .env.example .env
   # generate values:
   openssl rand -base64 32 | tr -d '/+='      # PROXY_PASS
   echo "$(openssl rand -hex 8).localhost"    # PROBE_DOMAIN
   nano .env
   docker compose up -d --build
   docker compose logs -f caddy               # wait for "certificate obtained successfully"
   ```

4. **Verify** (from any machine, replace the placeholders):

   ```sh
   # 1. The site is served
   curl -I https://DOMAIN

   # 2. Without credentials the proxy is invisible: curl reports
   #    "CONNECT tunnel failed, response 308" (the file server's redirect), never 407
   curl -v -x https://DOMAIN:443 https://ifconfig.me

   # 3. With credentials you get the VPS IP
   curl -x https://DOMAIN:443 -U 'USER:PASS' https://ifconfig.me

   # 4. Only the probe domain returns 407
   curl -I -x https://DOMAIN:443 http://PROBE_DOMAIN
   ```

Updating the site doesn't need a restart. Changes to `Caddyfile` or `.env` need `docker compose up -d --force-recreate`. To pick up a newer Caddy or plugin, run `docker compose build --pull --no-cache && docker compose up -d`.

## FoxyProxy

1. Add a proxy: type **HTTPS**, hostname `DOMAIN`, port `443`, plus your username and password.
2. Use **pattern** mode and add only the sites you need through the proxy, rather than "proxy everything". Normal browsing volume to the VPS looks less unusual. Add any work or internal domains as direct.
3. **Chrome:** after enabling the proxy, open `http://PROBE_DOMAIN` once. That's the only URL that answers with `407`, which makes the browser send your credentials. They're cached until the browser restarts, so repeat after a restart.
   **Firefox:** FoxyProxy usually sends the credentials up front. If proxied sites don't load, do the same probe-domain step.

## Known limits

- **Tunneled TLS can be spotted by traffic analysis.** Sites you open through the proxy run their own TLS handshake inside the outer TLS connection. The sizes and timing of those records follow a recognizable pattern, which DPI can learn to detect. A plain browser setup has no padding against this. The [NaiveProxy client](https://github.com/klzgrad/naiveproxy) does, and works with this server unchanged: run `naive --listen=socks://127.0.0.1:1080 --proxy=https://USER:PASS@DOMAIN` and point FoxyProxy at SOCKS5 `127.0.0.1:1080`.
- **The VPS IP or its hosting provider's network can be blocked or throttled outright**, regardless of how the traffic looks. This setup doesn't protect against that.
