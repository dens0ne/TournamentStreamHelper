# Remote Scoreboard Access

TSH's web server (Flask + Flask-SocketIO, see `src/TSHWebServer.py`) binds to
`0.0.0.0` on your LAN and serves the scoreboard control page at `/scoreboard`.
That's reachable to any device on the same network, but not from the public
internet. This doc covers exposing it remotely with
[Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/),
so a remote scorekeeper can open a URL in their browser and control your
scoreboard.

**Important:** TSH's web server has no built-in authentication or
authorization. Anyone who has the URL can view and control the scoreboard.
Pick the option below that matches how much you need to lock that down.

## Find your local port

TSH's web server port is `general.webserver_port` in `user_data/settings.json`
(default: `5500`). If that key isn't present, you're on the default. Check:

```bash
grep webserver_port user_data/settings.json
```

The commands below assume `5500` — substitute your actual port.

## Required setting: disable automatic score update blocking

In TSH's **Settings** window, uncheck *"Disable automatic score updating for
the scoreboard"* (`general.disable_scoreupdate` in `user_data/settings.json`).
If this is checked (`true`), the `/scoreboard` page's "Submit Changes" button
(and any other web-based score push) will return "OK" with no error, but the
score will silently fail to update on the local scoreboard widget — see
`TSHScoreboardWidget.py:1305`. This applies regardless of whether you're
using the scoreboard locally, over LAN, or through a tunnel.

## Option A — Quick Tunnel (free, fastest, no login wall)

Best for: short-lived use (e.g. one tournament weekend) where you're fine
sharing a one-off link directly with trusted people.

1. Install `cloudflared`:
   - Windows: `winget install --id Cloudflare.cloudflared`, or download from
     [github.com/cloudflare/cloudflared/releases](https://github.com/cloudflare/cloudflared/releases)
   - macOS: `brew install cloudflared`
   - Linux: see the [install docs](https://pkg.cloudflare.com/)
2. Make sure TSH is running (so its web server is listening).
3. Run, keeping this terminal open the whole time you need remote access:
   ```bash
   cloudflared tunnel --url http://localhost:5500
   ```
4. Cloudflare prints a random URL like
   `https://some-random-words.trycloudflare.com`. Share
   `<that URL>/scoreboard` with whoever needs remote access.
5. To stop: close the terminal, or `Ctrl+C`. The URL dies with it — starting
   it again later generates a **new** random URL.

No Cloudflare account, domain, or cost required. The trade-off: no login
gate, and the URL isn't stable across restarts.

## Option B — Named Tunnel + login gate (persistent URL, requires a domain)

Best for: ongoing use where you want a stable URL and an actual login
screen in front of it, not just an unlisted link.

**Requirements:** a domain added to Cloudflare (Cloudflare itself is free;
owning the domain is the only real cost, typically ~$10-15/year if you don't
already have one). Cloudflare Access is free for up to 50 users.

1. Add your domain to Cloudflare: [dash.cloudflare.com](https://dash.cloudflare.com/)
   → "Add a domain" → follow the nameserver-change steps at your registrar.
2. In the [Zero Trust dashboard](https://one.dash.cloudflare.com/) →
   **Networks → Tunnels → Create a tunnel** → choose "Cloudflared" → name it.
3. Copy the install command it gives you (includes a token) and run it on
   the machine running TSH. On Windows this installs and starts a service:
   ```
   cloudflared service install <token>
   ```
4. Back in the tunnel's configuration page, add a **Public Hostname**:
   - Subdomain/domain: whatever you want, e.g. `scoreboard.yourdomain.com`
   - Service type: `HTTP`
   - URL: `localhost:5500`
   - Saving this creates the DNS record automatically.
5. Add the login gate: **Access → Applications → Add an application** →
   "Self-hosted" → point it at the same hostname → add a policy (e.g. allow
   a specific list of emails, with one-time-PIN login, or SSO if you have
   it). Now visiting `https://scoreboard.yourdomain.com/scoreboard` prompts
   for login before reaching TSH at all.

To remove the service later (requires an elevated/Administrator terminal on
Windows):

```
"C:\Program Files (x86)\cloudflared\cloudflared.exe" service uninstall
```

## Which to pick

| | Quick Tunnel | Named Tunnel + Access |
|---|---|---|
| Cost | Free | Free (+ domain if you don't have one) |
| Setup time | ~2 minutes | ~15-20 minutes |
| URL stability | New random URL every restart | Permanent |
| Login gate | None — link is the only barrier | Yes, real auth (email/SSO) |
| Good for | One-off / short events | Recurring use, less trusted link |
