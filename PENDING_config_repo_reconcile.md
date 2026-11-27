# Pending — reconcile `homelab-configs` repo against live `/etc`

Logged 2026-08-03. Not urgent, not touched. Carl to action when convenient.

## 1. The repo is missing the cloudflared http2 fix (the real one)

`/home/pi/homelab-configs/cloudflared/config.yml` does **not** contain the block that
**is** live in `/etc/cloudflared/config.yml`:

```yaml
protocol: http2
retries: 5
grace-period: 30s
```

Confirmed by `diff` on 2026-08-03 — those are the only lines the repo lacks (`3,6d2`).

**Why this matters:** that block is the fix for the QUIC edge-drop outages that were
causing the Uptime-Kuma "everything down" waves. Restoring cloudflared from the
checked-in config would silently reintroduce the outage. The repo's most recent
commit is `43fda80 fix: stop Kuma paging on transient tunnel reconnects` — so the
commit message claims the fix while the file does not carry it. Worth a look at
whether that commit landed partially.

## 2. Four changes from 2026-08-03 are live but not in the repo

Made while standing up the `sk.highcarlsagan.dev` guest tier for Saurabh:

| File | Change |
|---|---|
| `/etc/authelia/users.yml` | added user `saurabh`, group `sk` |
| `/etc/authelia/configuration.yml` | 2 `access_control` rules: allow `group:sk` on `sk.` (one_factor), then explicit `deny` for `group:sk` on `*.highcarlsagan.dev` — inserted **above** the wildcards, first match wins |
| `/etc/caddy/Caddyfile` | new `sk.highcarlsagan.dev` block, `root * /srv/sk` |
| `/etc/cloudflared/config.yml` | ingress entry for `sk.highcarlsagan.dev` |

Backups of all four exist on the Pi as `<file>.bak-20260803`.

**Note if committing `users.yml`:** it contains argon2id password hashes. They are
hashes, not plaintext, but check whether that file is already tracked and whether
the repo is private before adding it.

## 3. Two smaller things spotted, also untouched

- `/etc/caddy/Caddyfile` is not `caddy fmt`-clean — `caddy validate` warns about
  line 2. Cosmetic; pre-existing.
- `/etc/caddy/` now holds **ten** `Caddyfile.bak*` files going back to April 2026.
  Cleanup candidate; already flagged once in the pi-pages notes.

## Suggested order

1. Work out why `43fda80` didn't carry the http2 lines — that's the one with a real
   failure mode behind it.
2. Sync the four `sk` changes.
3. Prune the `.bak` pile.
