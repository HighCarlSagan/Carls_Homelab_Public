# Architecture

A deeper look at how the homelab is wired together — the design decisions, the trade-offs, and the reasoning behind each component.

---

## The CGNAT Problem

Many residential ISPs (especially in regions with IPv4 exhaustion) put customers behind Carrier-Grade NAT. You get a private 100.64.0.0/10 address from the ISP, not a real public IP. Port forwarding is impossible — there's no port to forward *to*.

Traditional homelab guides assume you have a public IP and can forward port 443 on your router. Without that assumption, you have three real options:

1. **Rent a VPS** with a public IP and reverse-proxy through it. Works, but costs money and your traffic is on someone else's server.
2. **Use a tunnel service** that maintains an outbound connection from your homelab to a cloud edge. Cloudflare Tunnel does this for free.
3. **Use a mesh VPN** so all your devices reach the homelab as if on the same LAN. Tailscale does this for free.

This stack uses both **(2) and (3)** — Cloudflare Tunnel for *public* access (anyone on the internet hitting your subdomains) and Tailscale for *private* access (your own devices reaching SSH, internal admin pages, etc.).

The combination means **zero open inbound ports on your router**. CGNAT becomes a non-issue.

---

## Why Cloudflare Tunnel for public traffic

Cloudflare runs `cloudflared` as a systemd service on the Pi. It maintains four outbound persistent connections to Cloudflare's edge. When someone hits `ha.yourdomain.tld` from the internet, the request goes:

```
Browser → Cloudflare edge → existing tunnel connection → cloudflared on Pi → local services
```

The Pi never accepts an inbound connection from the internet. Your router never opens a port. Cloudflare handles TLS termination at their edge, plus DDoS protection, CDN caching for static assets, and bot protection — all on the free tier.

The downside: your traffic transits Cloudflare's network. They could in theory log and inspect it. For a personal homelab that's an acceptable trade. For anything more sensitive, you'd want self-hosted alternatives.

---

## Why Tailscale for private traffic

Tailscale is a managed WireGuard mesh. Every device you join becomes a peer on a private network with stable IPs (`100.x.x.x` range) regardless of where they physically sit on the internet. Devices can reach each other directly when possible, falling back through Tailscale's DERP relays when NAT prevents direct connections.

For the homelab, Tailscale handles:

- **SSH access** — only reachable on the Tailscale IP, not exposed to the public internet
- **Pi-hole DNS for all my devices** — even when traveling, my phone uses my Pi as its DNS server, blocking ads everywhere
- **Internal admin pages** — anything I don't want to expose publicly even with 2FA

The Tailscale IP of each device is stable forever (until you tell Tailscale otherwise). LAN IPs come and go with DHCP; Tailscale IPs don't. So config files reference Tailscale IPs, not LAN ones.

---

## Why Caddy

Three reasons over nginx or Traefik:

1. **Automatic HTTPS** — Caddy handles Let's Encrypt cert provisioning and renewal automatically with zero config. Or, with the `caddy-dns/cloudflare` module, it uses DNS-01 challenge via your Cloudflare token, which works for wildcard certs and doesn't need port 80.
2. **Readable config syntax** — Caddyfile is dramatically simpler than nginx config. A reverse-proxy-with-Authelia block is four lines.
3. **forward_auth pattern with Authelia is first-class** — Caddy ships clean primitives for the Authelia integration pattern.

The trade-off is fewer plugins and a smaller community than nginx. For a personal homelab with a dozen services, that's fine.

Caddy here is built with the `caddy-dns/cloudflare` module so it can use DNS-01 for wildcard certs. The module has [an upstream bug](#caddy-token-regex-patch) that required a local patch — see `docs/lessons-learned.md`.

---

## Why Authelia

For 2FA gating I considered Authelia, Authentik, and Keycloak. Authelia won on:

- **Lightweight** — single Go binary, file-based user store, no Postgres required
- **forward_auth pattern** — Caddy's `forward_auth` directive integrates cleanly
- **TOTP support out of the box** — works with Google Authenticator, Aegis, etc.

The user store (`users.yml`) is a flat YAML file with bcrypt password hashes and TOTP secrets. Fine for one or two users. For an org-scale setup you'd want LDAP or a real database.

**Access control rules** are evaluated in order. The current setup:

```yaml
access_control:
  default_policy: deny
  rules:
    - domain: "projects.yourdomain.tld"
      policy: bypass            # public, no auth
    - domain: "*.yourdomain.tld"
      policy: two_factor        # everything else needs 2FA
```

Specific bypass rules go first, the catch-all goes last, and `default_policy: deny` ensures anything you forget is locked by default.

---

## Why Pi-hole over local /etc/hosts

For ad-blocking, you could just maintain a blocklist in `/etc/hosts` on each device. Pi-hole gives you:

- **One place to update blocklists** — change it on the Pi, everyone benefits
- **Coverage for devices you can't edit** — phones, TVs, IoT junk
- **DNS query logs** for debugging what's actually getting blocked or breaking
- **Per-client policies** (advanced — not currently used)

Pi-hole here runs in Docker (more on why in [`docs/lessons-learned.md`](lessons-learned.md)). It listens on the Pi's LAN IP for queries from local devices, and Tailscale's "DNS override" feature lets it serve as the DNS for all mesh devices too — including when you're on cellular.

---

## Why MQTT (Mosquitto)

For ESP32 devices to talk to Home Assistant, MQTT is the cleanest pattern. Mosquitto is the default broker — tiny, fast, well-supported.

The integration works via **Home Assistant MQTT auto-discovery**. ESP32 publishes a JSON config message on a discovery topic when it boots; HA picks it up and creates the device + entities automatically. No YAML editing on the HA side.

The full discovery pattern is in [`docs/adding-a-service.md`](adding-a-service.md) — it scales to dozens of devices without any config bloat.

---

## Why Docker for some services and systemd for others

Three rules of thumb that drove the choice:

1. **Use native install** when the service has a stable binary or apt package, runs as a long-lived process, and benefits from systemd's logging/restart behavior. Caddy, Authelia, Mosquitto, cloudflared, Filebrowser → native.
2. **Use Docker** when the service is heavy, has finicky dependencies, or the native installer is broken. Pi-hole, Home Assistant, Cloudflare DDNS, Uptime Kuma → Docker.
3. **Use systemd user services** for personal apps with no system-wide install needed. Passives Tracker runs as a Python app under the `pi` user with `loginctl enable-linger` so it survives reboots.

The Docker containers are run with `--restart unless-stopped` so they come up on boot. Mixed approach keeps everything observable through `systemctl` and `docker ps`.

---

## DNS Architecture

There are three DNS layers, and getting them straight matters:

1. **Cloudflare authoritative DNS** — your domain points here. CNAMEs for each subdomain point to `<tunnel-id>.cfargotunnel.com` so traffic flows through your tunnel.
2. **Pi-hole** — listens on the Pi (LAN + Tailscale). All your devices use this as their DNS server. It blocks ads, then forwards legitimate queries upstream.
3. **Upstream resolvers** — Pi-hole forwards to your chosen upstream (Cloudflare 1.1.1.1, Quad9, or similar) for actual resolution.

**Tailscale DNS override** is the key trick that makes this work everywhere. In the Tailscale admin console, you set "Override local DNS" with your Pi's Tailscale IP. Every device on your tailnet now uses Pi-hole regardless of physical location — phone on cellular, laptop at a coffee shop, all of it.

---

## TLS Termination Strategy

TLS happens at **two** layers, which is occasionally surprising:

1. **Cloudflare ↔ Browser**: Cloudflare-issued cert, automatic, "Full (strict)" mode in CF settings.
2. **Cloudflared ↔ Caddy**: Cloudflared connects to Caddy on `https://localhost:443`. Caddy presents its own Let's Encrypt cert (or the wildcard from the cloudflare DNS-01 challenge). Cloudflared trusts the cert because of `originServerName` in the tunnel config.

Without `noTLSVerify: true` or `originServerName` set correctly, cloudflared will reject the cert as not matching the hostname (since it's connecting to `localhost` but expects `ha.yourdomain.tld`). Both options work; `originServerName` is cleaner.

---

## Backup Strategy

Three layers of backup:

1. **Daily config snapshot** — `homelab-backup.sh` runs at 2am via cron, tarballs all critical config dirs to `/srv/filebrowser/backups/`. Keeps the last 7. Accessible via Filebrowser at `files.yourdomain.tld/files/backups/`.
2. **Git** — every config is committed to a private git repo. Pi auto-pushes after every change via the workflow in `lessons-learned.md`.
3. **Bitwarden** — secrets (Authelia users.yml, Cloudflare token, tunnel JSON) are stored as secure notes in Bitwarden, not in any git repo.

Recovery from total loss = (1) flash new SD, (2) clone git repo, (3) run restore.sh, (4) paste secrets from Bitwarden. Estimated 30–45 minutes end-to-end.

---

## What's Deliberately Excluded

Decisions I made *not* to include, with reasoning:

- **Self-hosted email** — running a mail server is a nightmare. Cloudflare Email Routing forwards incoming mail to Gmail for free. Outgoing is Gmail with the "From" header set to my custom domain via an app password.
- **Self-hosted git** — GitHub does this well, no reason to duplicate. A future plan is to add a backup remote (Codeberg or Gitea on the Arch PC) so I'm not single-cloud.
- **Photos/media hosting** — could add Immich or PhotoPrism, but Google Photos handles this fine for now and the Pi 4 isn't ideal for that workload.
- **Self-hosted password manager** — Bitwarden offers a free hosted tier and Vaultwarden adds operational complexity. Maybe later.
- **Off-Pi monitoring** — Uptime Kuma is monitoring itself, which is a smell. Should add a second monitor (something like a free Healthchecks.io account) that pings the Pi from outside.

---

## Caddy Token Regex Patch

The `caddy-dns/cloudflare` module validates Cloudflare API tokens against a regex that caps at 50 characters. Cloudflare started issuing 52-character `cfut_` tokens in 2024. The fix is a one-character change:

```bash
sed -i 's/{35,50}/{35,100}/g' cloudflare.go
```

Then rebuild Caddy with `xcaddy build --with github.com/caddy-dns/cloudflare=.` pointing at the patched local copy. The compiled binary replaces `/usr/bin/caddy`. The full procedure is in [`docs/lessons-learned.md`](lessons-learned.md) and embedded in `restore.sh.example`.
