# Carls Homelab — Public Reference

![Status](https://img.shields.io/badge/status-running-brightgreen)
![Hardware](https://img.shields.io/badge/host-Raspberry%20Pi%204-c51a4a)
![Network](https://img.shields.io/badge/network-CGNAT%20bypassed-blue)
![Auth](https://img.shields.io/badge/auth-Authelia%202FA-orange)
![License](https://img.shields.io/badge/license-MIT-green)

The public reference for my self-hosted homelab — a Raspberry Pi 4 sitting on a CGNAT'd home connection, running every service I depend on, accessible from anywhere with 2FA, and reproducible from scratch with a single script.

This repo is the sanitized public blueprint. It documents how the stack is put together, the design decisions behind it, the lessons learned along the way, and includes generic templates of every config file. The operational backup of my actual instance (with real identifiers) lives in a private companion repo.

---

## Why I Built This

Self-hosting is the only way I can keep my data on my own hardware, on my own terms. The major reasons:

- **Self-sovereign data.** My files, my smart-home telemetry, my parts inventory, my notes — none of it on a third-party cloud or behind a corporate paywall.
- **Beat CGNAT elegantly.** My ISP doesn't give me a real public IP, so traditional port forwarding is impossible. I wanted to solve this cleanly via outbound-only tunnels rather than rented VPSes or relays I don't control.
- **Single pane for personal infrastructure.** Ad-blocking DNS, smart-home automation, file sync, MQTT for ESP32 devices, status monitoring — all in one place I control.
- **Learn Linux server admin properly.** Caddy, systemd, Docker, Tailscale, Cloudflare Tunnel, Authelia — wanted hands-on experience configuring each rather than letting a turn-key distro hide it from me.
- **Reproducible from scratch.** If the SD card dies tomorrow, one `restore.sh` rebuilds everything in under an hour.
- **No paid services.** Tailscale free tier, Cloudflare free tier, no recurring cost beyond the Pi itself.

The setup is meant to be useful as a reference — if you're building your own and any of the above resonates, the architecture and templates in this repo should give you a strong starting point.

---

## Initial Requirements

| # | Requirement | Outcome |
|---|---|---|
| 1 | Always-on, low-power server | Raspberry Pi 4 4GB |
| 2 | Zero open inbound ports on the router | Tailscale + Cloudflare Tunnel (both outbound-only) |
| 3 | 2FA on every public-facing service | Authelia + TOTP on every subdomain except portfolio + resume |
| 4 | Reproducible from scratch with one script | `restore.sh` rebuilds the stack |
| 5 | Ad-blocking DNS for all my devices | Pi-hole with Tailscale DNS override |
| 6 | Smart-home automation hub | Home Assistant |
| 7 | File sync / remote file access | Filebrowser |
| 8 | MQTT broker for ESP32 fleet | Mosquitto |
| 9 | Free or near-free tier on all paid services | Cloudflare free, Tailscale free |
| 10 | Service uptime monitoring | Uptime Kuma |
| 11 | Daily automated config backup | Cron job → archive on Filebrowser |
| 12 | Custom domain email without running a mail server | Cloudflare Email Routing → Gmail |

---

## Architecture

```
                INTERNET (CGNAT — no inbound possible)
                            │
                            ▼
                ┌──────────────────────┐
                │     CLOUDFLARE       │
                │   DNS + Tunnel edge  │
                └──────────┬───────────┘
                           │  (outbound tunnel, no open ports)
                           ▼
                ┌──────────────────────┐         ┌─────────────────┐
                │  RASPBERRY PI 4      │◀───────▶│   TAILSCALE     │
                │  (always-on server)  │  mesh   │   coordination  │
                └──────────┬───────────┘         └─────────────────┘
                           │
                  ┌────────┴────────┐
                  ▼                 ▼
            ┌──────────┐      ┌──────────┐
            │  CADDY   │      │ PI-HOLE  │
            │ reverse  │      │   DNS    │
            │  proxy   │      │  sinkhole│
            └────┬─────┘      └──────────┘
                 ▼
            ┌──────────┐
            │ AUTHELIA │   2FA gate on every protected subdomain
            └────┬─────┘
                 ▼
       ┌─────────┴─────────┬──────────────┬──────────────┐
       ▼                   ▼              ▼              ▼
  Home Assistant     Filebrowser     Mosquitto       (other
   (smart home)     (file access)      (MQTT)        services)
```

**Traffic flow for an external visitor:**
```
Browser → Cloudflare CDN → Cloudflare Tunnel → cloudflared (Pi)
       → Caddy (TLS termination) → Authelia (2FA check)
       → Backend service
```

**Zero open inbound ports.** Both Tailscale and Cloudflare Tunnel are outbound-initiated. CGNAT is not a limitation with this stack.

Full architecture details: [`docs/architecture.md`](docs/architecture.md)

---

## Stack

| Category | Service | Method |
|---|---|---|
| Reverse proxy | Caddy (custom patched build) | Native systemd |
| Authentication | Authelia | Native systemd |
| Outbound tunnel | cloudflared | Native systemd |
| Mesh VPN | Tailscale | Native systemd |
| DNS sinkhole / ad-blocking | Pi-hole | Docker |
| Smart-home hub | Home Assistant | Docker |
| MQTT broker | Mosquitto | Native systemd |
| File access | Filebrowser | Native systemd |
| Uptime monitoring | Uptime Kuma | Docker |
| DDNS | Cloudflare DDNS | Docker |
| Parts inventory | Passives Tracker (custom) | Python systemd user service |
| Email routing | Cloudflare Email Routing | Cloudflare-managed |

All non-Caddy services bind to `127.0.0.1` only. Caddy is the only process facing the network. Everything else is reachable solely through the reverse proxy.

---

## Repository Structure

```
Carls_Homelab_Public/
├── README.md                                    # This file
├── docs/
│   ├── architecture.md                          # Deep architecture writeup
│   ├── lessons-learned.md                       # Hard-won gotchas worth capturing
│   └── adding-a-service.md                      # 5-step pattern for new services
├── templates/
│   ├── Caddyfile.example                        # Reverse proxy + 2FA gating
│   ├── cloudflared-config.yml.example           # Tunnel ingress rules
│   ├── authelia-configuration.yml.example       # Auth + access control
│   ├── restore.sh.example                       # Full rebuild script
│   └── homelab-backup.sh.example                # Daily config backup script
├── LICENSE                                       # MIT
└── .gitignore
```

---

## Quick Start (for your own homelab)

You can't copy-paste this and expect it to run — your domain, your tokens, your hostnames are different. Treat the templates as a starting point and walk through them:

1. **Provision a Pi 4** (or any always-on Linux box) with a static LAN IP
2. **Buy a domain on Cloudflare** (or transfer one in) — the free Cloudflare tier is sufficient
3. **Set up a Cloudflare Tunnel** following the cloudflared documentation
4. **Install Tailscale** and join your devices to the mesh
5. **Adapt `Caddyfile.example`** with your domain and your service ports
6. **Adapt `cloudflared-config.yml.example`** with your tunnel ID and ingress rules
7. **Configure Authelia** following the upstream docs — `configuration.yml.example` is the protected setup pattern
8. **Adapt `restore.sh.example`** with your username, paths, and Cloudflare DDNS domain
9. **Set up daily backup** with `homelab-backup.sh.example` as a cron job

Detailed walkthrough: [`docs/adding-a-service.md`](docs/adding-a-service.md) for the per-service pattern once the base stack is up.

---

## Lessons Learned

A few things that bit me hard enough to be worth writing down. Full list with details in [`docs/lessons-learned.md`](docs/lessons-learned.md).

- **Caddy `caddy-dns/cloudflare` rejects newer 52-character Cloudflare API tokens** — the upstream regex caps at 50 chars. Solved by building Caddy with a locally patched module.
- **Authelia `apt` repository is dead** — install from GitHub binary releases. Don't use `apt install authelia`.
- **Pi-hole's native installer is incompatible with Kitty terminal** — hardcoded ncurses dialogs. Always use the Docker image instead.
- **Authelia `forward_auth` needed `trusted_proxies static private_ranges`** in the Caddy server block — without it, headers don't propagate correctly through Cloudflare → Caddy → backend.
- **Mosquitto's password file gets created as root and Mosquitto can't read it** — always `chown mosquitto:mosquitto /etc/mosquitto/passwd` after `mosquitto_passwd`.
- **Home Assistant returns 400 behind a reverse proxy** until you add `use_x_forwarded_for: true` and `trusted_proxies` at the root level of `configuration.yaml`.
- **Never use `nano` on the Pi** if you SSH from Kitty — `TERM=xterm-kitty` breaks ncurses. Use `sed` for config edits, or set `TERM=xterm-256color` before SSHing.

---

## Disaster Recovery

The stack is designed to survive an SD card failure with a 30–45 minute recovery time:

1. Flash fresh Pi OS Lite 64-bit
2. Clone the **private** companion repo (which contains the actual configs)
3. Run `restore.sh` — installs all dependencies, copies configs, starts services
4. Manually restore secrets from a password manager (Bitwarden in my case)
5. Re-authenticate Tailscale and re-register TOTP

Secrets that live outside git (always in a password manager, never in either repo):

- `users.yml` for Authelia (contains password hashes and TOTP secrets)
- Cloudflare API token
- Tunnel credentials JSON
- Mosquitto password file
- Pi-hole admin password

---

## Adding a New Service

Every new service follows the same 5-step pattern:

1. Install the service, bind it to `127.0.0.1` on a free port
2. Add a Caddy block to `Caddyfile` (with or without `import authelia`)
3. Add a Cloudflare DNS CNAME record pointing to the tunnel
4. Add an ingress rule to `cloudflared/config.yml`
5. Add an Uptime Kuma monitor, commit configs to git

Full walkthrough with code: [`docs/adding-a-service.md`](docs/adding-a-service.md)

---

## What's In, What's Out

**What this repo is:**
- A reference for how the stack fits together
- Sanitized templates of every config file
- The lessons learned that made the stack actually work

**What this repo is NOT:**
- A runnable installer (the real values are in a separate, private repo)
- A blueprint for a specific architecture you should copy 1:1 — your needs will differ
- Production-grade, audited, security-reviewed software — it's a personal homelab

---

## Related Projects

- [PC_and_Light_Controller](https://github.com/HighCarlSagan/PC_and_Light_Controller) — ESP32-based Telegram controller for desktop UPS button and room lighting. Talks MQTT to the Mosquitto broker in this stack.
- [m-hypr-config](https://github.com/HighCarlSagan/m-hypr-config) — my Hyprland desktop config on the Arch PC that integrates with this homelab over Tailscale.

The Passives Tracker (a custom electronics parts inventory) lives in this homelab as `passive.highcarlsagan.dev`. It's a small Python/SQLite app that runs as a systemd user service — its source isn't broken out into a separate public repo (yet).

---

## License

MIT. See [LICENSE](LICENSE). Use, fork, adapt freely.

---

## Author

**Mayank S (HighCarlSagan)** — [GitHub @HighCarlSagan](https://github.com/HighCarlSagan) · [highcarlsagan.dev](https://highcarlsagan.dev)


Built and tuned over many late nights in Bangalore.
