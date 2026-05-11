# Adding a New Service

Every new service follows the same 5-step pattern. Once you've internalized it, adding a service takes about 10 minutes.

This guide assumes the base stack (Caddy, Authelia, cloudflared, Tailscale) is already up and running. If not, start with the main [README](../README.md).

---

## The Pattern

1. **Install the service** and bind it to `127.0.0.1` on a free port
2. **Add a Caddy block** to your `Caddyfile`
3. **Add a Cloudflare CNAME** record
4. **Add an ingress rule** to `cloudflared/config.yml`
5. **Add Uptime Kuma monitor + commit configs**

That's it. No router config, no firewall changes, no DNS propagation waits — all of that is handled by the tunnel.

---

## Step 1: Install the service

Whatever the service is (Docker container, native binary, Python app), bind it to `127.0.0.1` on a port you've picked. Recommended free port ranges:

| Range | Use |
|---|---|
| 3000+ | UI services |
| 4000+ | dev / experimental |
| 8000–8999 | misc HTTP backends |

Check your `Caddyfile` for what's already taken before picking one.

**Critical:** services must bind to `127.0.0.1`, not `0.0.0.0`. If they bind to all interfaces, anyone on your LAN can hit them directly, bypassing Authelia. Make sure the listen address is loopback-only.

**Docker example** — note the `127.0.0.1:` prefix in the port mapping:

```bash
docker run -d \
  --name myapp \
  --restart unless-stopped \
  -p 127.0.0.1:8082:8080 \
  myorg/myapp:latest
```

**Native systemd service example** — configure the app to bind loopback in its own config (most apps have a `listen_address` or similar setting).

---

## Step 2: Add a Caddy block

Edit `/etc/caddy/Caddyfile` and add one of these patterns. Always use `sed` for edits, not `nano` (see [lessons-learned.md](lessons-learned.md)).

**Pattern A: Protected service (requires Authelia 2FA)** — the default for almost everything:

```caddy
myapp.yourdomain.tld {
  import tls_cloudflare
  import authelia
  reverse_proxy http://127.0.0.1:8082
}
```

**Pattern B: Public service (no auth)** — only for things you genuinely want anyone to access:

```caddy
myapp.yourdomain.tld {
  import tls_cloudflare
  reverse_proxy http://127.0.0.1:8082
}
```

**Pattern C: Static files** — for serving a folder of HTML/assets:

```caddy
myapp.yourdomain.tld {
  import tls_cloudflare
  import authelia
  root * /srv/myapp
  file_server
  encode gzip
}
```

Reload Caddy after editing:

```bash
sudo systemctl reload caddy
```

If Caddy fails to reload, run `sudo caddy validate --config /etc/caddy/Caddyfile` to see the error before retrying.

---

## Step 3: Add a Cloudflare DNS CNAME record

In the Cloudflare dashboard for your domain:

- DNS → Records → Add record
- **Type:** CNAME
- **Name:** `myapp` (will become `myapp.yourdomain.tld`)
- **Target:** `<your-tunnel-id>.cfargotunnel.com`
- **Proxy status:** Proxied (orange cloud — gives you DDoS protection + CDN)
- **TTL:** Auto

Save. DNS propagation through Cloudflare is near-instant when proxied.

---

## Step 4: Add ingress rule to cloudflared

Edit `/etc/cloudflared/config.yml`. Add a new entry to the `ingress:` list — anywhere **before** the final catch-all `service: http_status:404`:

```yaml
- hostname: myapp.yourdomain.tld
  service: https://localhost:443
  originRequest:
    noTLSVerify: true
    originServerName: myapp.yourdomain.tld
```

The `originServerName` matters — see [lessons-learned.md](lessons-learned.md) for why.

Restart cloudflared:

```bash
sudo systemctl restart cloudflared
```

The four tunnel connections will re-establish in a few seconds. Check status:

```bash
sudo systemctl status cloudflared
sudo journalctl -u cloudflared -n 20
```

---

## Step 5: Add monitoring + commit

**Uptime Kuma monitor:** open `status.yourdomain.tld` → Add Monitor → HTTP(s) → `http://127.0.0.1:8082` (the local backend, not the public URL — this checks the actual service, not the proxy chain).

**Commit the changes to git:**

```bash
cd ~/homelab-configs
sudo cp /etc/caddy/Caddyfile caddy/
sudo cp /etc/cloudflared/config.yml cloudflared/
git add -A
git commit -m "add: myapp service on port 8082"
git push
```

That's it. Hit `https://myapp.yourdomain.tld` from anywhere, log in with Authelia, and you're routed to the local service.

---

## Adding an ESP32 / IoT Device

ESP32 devices skip steps 2–4 (they don't get a subdomain — they talk to Mosquitto MQTT directly) and instead follow this pattern:

### Step 1: Flash with MQTT-aware firmware

Use PlatformIO or Arduino IDE. Required library: `knolleary/PubSubClient`.

Configure in your firmware:
- WiFi SSID + password
- MQTT broker IP (your Pi's LAN IP)
- MQTT user + password (matching your Mosquitto passwd file)
- Unique device ID — convention: `esp32_<location>_<number>` (e.g. `esp32_workshop_01`)

### Step 2: Publish discovery payload on boot

On MQTT connect, publish **once** with `retain: true` to:

```
Topic: homeassistant/sensor/<device_id>_<sensor>/config
```

Minimal JSON payload:

```json
{
  "name": "Temperature",
  "uniq_id": "esp32_node01_temp",
  "stat_t": "homelab/esp32_node01/temperature",
  "unit_of_meas": "°C",
  "dev_cla": "temperature",
  "dev": {
    "ids": ["esp32_node01"],
    "name": "Workshop Node 01",
    "mdl": "ESP32",
    "mf": "Espressif"
  }
}
```

The `retain: true` flag is critical — it means HA picks up the device even if HA restarts after the ESP32 has already booted.

### Step 3: Home Assistant auto-discovers

After the ESP32 boots and publishes discovery, go to HA → Settings → Devices & Services → Devices. The device appears automatically with all its entities. No YAML editing needed.

### Step 4: Publish readings periodically

Publish plain string values to the state topic from your discovery payload:

```
Topic:   homelab/esp32_node01/temperature
Payload: "23.4"
```

HA reads and stores them automatically.

### Step 5: Add a command topic for actuators

For relays, switches, or anything HA should control, add a `cmd_t` to your discovery payload:

```json
"cmd_t": "homelab/esp32_node01/relay/set"
```

HA will publish "ON" or "OFF" to that topic. Subscribe to it in firmware and drive the GPIO accordingly.

---

## Gotchas

- **Device IDs must be unique per ESP32.** Duplicate IDs cause HA to merge two devices into one and break entity tracking. Stick to a naming convention.
- **Retain the discovery payload.** Without `retain: true`, you lose the device every time HA restarts.
- **Publish state on a schedule, not on every loop tick.** Once every 30 seconds is plenty for most sensors. Drowning the broker in messages will hurt everything else on it.
- **Use the Tailscale IP of the Pi**, not its LAN IP, in firmware if you want devices to reach MQTT from anywhere. Tailscale IPs are stable; LAN DHCP isn't.
