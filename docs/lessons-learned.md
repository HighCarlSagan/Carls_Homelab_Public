# Lessons Learned

Things that bit me hard enough to be worth writing down. Every one of these cost real time to debug, and every one of them will catch the next person too.

---

## Caddy: `caddy-dns/cloudflare` rejects newer tokens

**Symptom:** Caddy fails to issue TLS certs via DNS-01 challenge. Logs show the Cloudflare module rejecting the API token as invalid format.

**Cause:** The upstream module validates tokens with a regex capped at 50 characters. Cloudflare started issuing 52-character `cfut_` prefix tokens in 2024.

**Fix:** Patch the regex locally and build Caddy with your patched module.

```bash
# Clone the module locally
git clone https://github.com/caddy-dns/cloudflare ~/caddy-cloudflare-patched
cd ~/caddy-cloudflare-patched

# Patch — extend max length from 50 to 100
sed -i 's/{35,50}/{35,100}/g' cloudflare.go

# Build a custom Caddy with the patched module
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
~/go/bin/xcaddy build --with github.com/caddy-dns/cloudflare=.

# Replace the system binary
sudo mv caddy /usr/bin/caddy
sudo setcap cap_net_bind_service=+ep /usr/bin/caddy
sudo systemctl restart caddy
```

**Do not run `apt install caddy`** after this — it will overwrite the patched binary.

---

## Authelia: `apt` repository is dead

**Symptom:** `apt install authelia` returns 404. `apt.authelia.com` is no longer reachable.

**Cause:** Upstream removed the apt repo in favor of GitHub binary releases.

**Fix:** Install from the GitHub release directly.

```bash
AUTHELIA_VERSION="4.39.16"   # check latest on the releases page
curl -fsSL "https://github.com/authelia/authelia/releases/download/v${AUTHELIA_VERSION}/authelia-linux-arm64.tar.gz" \
  -o /tmp/authelia.tar.gz
tar -xzf /tmp/authelia.tar.gz -C /tmp
sudo mv /tmp/authelia-linux-arm64 /usr/local/bin/authelia
sudo chmod +x /usr/local/bin/authelia
```

Then create a systemd unit pointing at `/usr/local/bin/authelia` with your config path. Remove `authelia` from `apt`'s sources list if you'd previously added it.

---

## Authelia: `forward_auth` needs `trusted_proxies`

**Symptom:** Authelia redirects loop, or downstream services don't receive `Remote-User` / `Remote-Groups` headers.

**Cause:** Without trusted proxy configuration, Authelia doesn't trust the headers from Caddy and refuses to issue forwarded auth headers.

**Fix:** Set Caddy's `trusted_proxies` at the global level:

```caddy
{
  servers {
    trusted_proxies static private_ranges
  }
}
```

`private_ranges` covers RFC1918 (10/8, 172.16/12, 192.168/16), `127.0.0.0/8`, and `100.64.0.0/10` (CGNAT and Tailscale). That's what you want for any setup where Caddy receives connections from cloudflared (loopback) and Tailscale peers.

---

## Pi-hole: Docker over native installer

**Symptom:** Native Pi-hole installer hangs forever or errors out cryptically when running over SSH from Kitty terminal.

**Cause:** The installer uses hardcoded ncurses dialogs that can't be suppressed with `--unattended` and behave badly on certain terminal types.

**Fix:** Always use Docker.

```bash
sudo docker run -d \
  --name pihole \
  --network host \
  --restart unless-stopped \
  -e TZ="Asia/Kolkata" \
  -e FTLCONF_webserver_port="8888" \
  -e FTLCONF_dns_listeningMode="all" \
  -e WEBPASSWORD="<your-pihole-password>" \
  -v /etc/pihole:/etc/pihole \
  -v /etc/dnsmasq.d:/etc/dnsmasq.d \
  pihole/pihole:latest
```

**Pi-hole v6 renamed several env vars from v5:**

| v5 | v6 |
|---|---|
| `pihole -w` | `pihole allow` |
| `WEB_PORT` | `FTLCONF_webserver_port` |
| `DNSMASQ_LISTENING` | `FTLCONF_dns_listeningMode` |

Don't blindly follow old guides.

---

## Mosquitto: passwd file ownership

**Symptom:** Mosquitto refuses MQTT connections with auth errors. Logs show "unable to open password file".

**Cause:** `mosquitto_passwd -c /etc/mosquitto/passwd <user>` creates the file as root. Mosquitto runs as the `mosquitto` user and can't read it.

**Fix:** After every `mosquitto_passwd` invocation (creation or update):

```bash
sudo chown mosquitto:mosquitto /etc/mosquitto/passwd
sudo systemctl restart mosquitto
```

Add this as the line immediately after every `mosquitto_passwd` in any script you write.

---

## Home Assistant: 400 Bad Request behind reverse proxy

**Symptom:** HA returns "400 Bad Request" when accessed via the reverse proxy, but works fine locally.

**Cause:** HA refuses requests from unknown proxies for security. Needs explicit `use_x_forwarded_for` and `trusted_proxies` config.

**Fix:** In `/etc/homeassistant/configuration.yaml`, at the **root level** (not nested under `homeassistant:`):

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - ::1
```

If you also access HA via Tailscale directly (bypassing Caddy), add your Pi's Tailscale IP and the Tailscale CGNAT range too:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - ::1
    - 100.64.0.0/10
```

---

## Pi: don't use `nano` over SSH from Kitty

**Symptom:** `nano` opens with "Error opening terminal: xterm-kitty" and refuses to start.

**Cause:** `nano` uses ncurses with a hardcoded terminfo lookup. Kitty's `TERM=xterm-kitty` isn't in Pi OS's default terminfo database.

**Fix:** Two options.

1. Use `sed` for config edits. Always. It scales better anyway.
2. If you must use a TUI tool, export `TERM=xterm-256color` before SSHing:
   ```bash
   TERM=xterm-256color ssh pi
   ```

Or install the kitty terminfo on the Pi:
```bash
ssh pi "infocmp" > /dev/null  # this should not error
# If it does, on the Pi:
sudo apt install kitty-terminfo
```

Pragmatically, just use `sed`. Future-you will thank you.

---

## Cloudflared: tunnel config needs `originServerName`

**Symptom:** Cloudflared logs show TLS verification failures connecting to Caddy. Subdomain requests return 502.

**Cause:** Cloudflared connects to `https://localhost:443`, but Caddy's cert is for `ha.yourdomain.tld`, not `localhost`. Cert validation fails.

**Fix:** In each ingress rule, set `originServerName` to match the hostname being requested:

```yaml
- hostname: ha.yourdomain.tld
  service: https://localhost:443
  originRequest:
    noTLSVerify: true
    originServerName: ha.yourdomain.tld
```

`noTLSVerify: true` is the brute-force option (works but skips validation entirely). `originServerName` is the clean option (validates against the right name).

---

## Cron: backup script silently fails when run by cron

**Symptom:** Backup script works when run manually. Doesn't work when run by cron, no error visible.

**Cause:** Cron runs with a minimal `PATH` (`/usr/bin:/bin`). Anything from `/usr/local/bin/` or `~/.local/bin/` isn't reachable. Also no `$HOME` defined for many tools.

**Fix:** Add a full `PATH` at the top of the crontab, and use absolute paths in the script:

```cron
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
SHELL=/bin/bash
0 2 * * * /home/pi/homelab-configs/homelab-backup.sh >> /var/log/homelab-backup.log 2>&1
```

The `>> log 2>&1` part is what saves your sanity — without it, cron emails you the output, and if mail isn't configured you never see it.

---

## Tailscale DNS override

**Symptom:** Pi-hole works perfectly on your LAN. As soon as your phone leaves WiFi, ad-blocking stops and you start seeing ads on cellular.

**Cause:** DHCP-assigned DNS only applies on the local network. Off-network, the device falls back to cellular carrier DNS.

**Fix:** In the Tailscale admin console, navigate to DNS → Nameservers → enable "Override local DNS" and set your Pi's Tailscale IP as the nameserver. Every device on your tailnet now uses Pi-hole regardless of physical location.

You also need Pi-hole to listen on the Tailscale interface, not just LAN:

```bash
docker exec pihole pihole-FTL --listen-all
```

Or set `FTLCONF_dns_listeningMode=all` in the Pi-hole Docker env.

---

## Authelia: TOTP enrollment is one-shot

**Symptom:** You set up Authelia, scanned the QR code, then lost your phone or wiped Google Authenticator. You can no longer log in.

**Cause:** Authelia stores the TOTP secret in `users.yml` and exposes the QR code only on first enrollment. After that, the enrollment flow is closed.

**Fix:** Two precautions worth taking ahead of time:

1. **Back up the TOTP secret** when you first enroll. The secret is the string that gets QR-encoded. Save it in Bitwarden alongside the password. You can re-enter it manually into any TOTP app.
2. **Or back up the Authenticator app itself.** Aegis (Android) lets you export an encrypted backup. Google Authenticator now supports Google account sync. Either way, have an escape hatch.

If you do lose access without either: SSH into the Pi, edit `users.yml`, clear the TOTP field for your user, and re-enroll on next login.

---

## Filebrowser DB lives outside config

**Symptom:** You restore your homelab from git, but Filebrowser has no users and no settings.

**Cause:** Filebrowser stores users, settings, and shares in a SQLite database (`filebrowser.db`), not in a config file. Restoring `filebrowser.json` doesn't restore the actual state.

**Fix:** Include `filebrowser.db` in the backup tarball. Or, accept that you re-init Filebrowser from scratch on restore:

```bash
filebrowser config init
filebrowser users add admin "<password>" --perm.admin
```

Then re-add shares manually. For a personal setup, re-init is usually faster than carrying the DB around.
