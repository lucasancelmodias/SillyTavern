# SillyTavern on Coolify, private over Tailscale

This guide deploys this fork on a VPS running [Coolify](https://coolify.io) v4, using
[`docker-compose.coolify.yml`](../docker-compose.coolify.yml), and makes it reachable only from
your own devices through [Tailscale](https://tailscale.com). Nothing about SillyTavern is exposed to
the internet.

```text
your laptop / phone (on your tailnet)
   │  https://sillytavern.<your-tailnet>.ts.net
   ▼
tailscale serve  (on the VPS host, terminates HTTPS)
   │  http://127.0.0.1:8001
   ▼
SillyTavern container :8000   (port published on the VPS loopback only)
```

A public-domain setup is covered at the end, in [Public domain instead of Tailscale](#public-domain-instead-of-tailscale).

## Contents

1. [Before you start](#before-you-start)
2. [Install Tailscale on the VPS](#1-install-tailscale-on-the-vps)
3. [Deploy in Coolify](#2-deploy-in-coolify)
4. [Publish it on your tailnet](#3-publish-it-on-your-tailnet)
5. [Set a password](#4-set-a-password)
6. [Lock down the VPS](#5-lock-down-the-vps)
7. [Optional: restrict tailnet access](#6-optional-restrict-who-on-the-tailnet-can-reach-it)
8. [Settings reference](#settings-reference)
9. [Public domain instead of Tailscale](#public-domain-instead-of-tailscale)
10. [Updating, backups, troubleshooting](#updating)

## Before you start

- **Coolify v4** installed on the VPS.
- **Docker Engine 28 or newer** on the VPS (`docker version`). Older versions let other machines on
  the same network segment reach ports published on `127.0.0.1`.
- **Memory:** the image build (npm install + webpack) peaks around 1 GB on top of Coolify's own
  ~2 GB. On a 4 GB server, add swap if builds get killed.
- A **Tailscale account**, with Tailscale installed on the devices you will use SillyTavern from.

## 1. Install Tailscale on the VPS

SSH into the VPS over its public IP and keep that session open until the end of
[step 5](#5-lock-down-the-vps).

**Install** with the official script, which adds Tailscale's apt repository and installs the
package:

```sh
curl -fsSL https://tailscale.com/install.sh | sh
```

<details>
<summary>Or add the apt repository by hand (Ubuntu 24.04)</summary>

```sh
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
sudo apt-get update && sudo apt-get install -y tailscale
```

For Debian 12, replace `ubuntu/noble` with `debian/bookworm`.
</details>

The package enables and starts the `tailscaled` service. Check with `systemctl status tailscaled`.

**Join your tailnet.** The hostname becomes the `https://<hostname>.<tailnet>.ts.net` address. Once
HTTPS is enabled it is published in public certificate-transparency logs, so pick something
non-sensitive:

```sh
sudo tailscale up --hostname=sillytavern
```

Open the login URL it prints and approve the machine. (For unattended setups you can instead
generate a non-ephemeral auth key in the admin console under **Settings → Keys** and run
`sudo tailscale up --auth-key=file:/root/ts-authkey --hostname=sillytavern`. Do not use an
*ephemeral* key for a server.)

**In the [admin console](https://login.tailscale.com/admin):**

1. **Machines** → `sillytavern` → **⋯** → **Disable key expiry**. Otherwise the VPS drops off the
   tailnet when its key expires (180 days by default).
2. **DNS** → make sure **MagicDNS** is on, then under **HTTPS Certificates** click **Enable HTTPS**.
   `tailscale serve` needs this for the `*.ts.net` certificate.

**Check it works:**

```sh
tailscale status     # lists your devices
tailscale ip -4      # the VPS's 100.x.y.z address
```

From your laptop, `tailscale ping sillytavern` should get replies.

Optional: `sudo tailscale set --auto-update` keeps the Tailscale client updated.

## 2. Deploy in Coolify

1. **New Resource → Private Repository (with GitHub App)**, pick this fork and the branch to deploy.
2. Set:
   - **Build Pack:** Docker Compose
   - **Base Directory:** `/`
   - **Docker Compose Location:** `/docker-compose.coolify.yml`
3. **Do not add a domain** to the `sillytavern` service. If Coolify generated one (for example an
   `sslip.io` address), delete it under **Domains**. Even if one slips through, the IP whitelist
   answers requests that arrive through Coolify's proxy with 403.
4. **Environment Variables:** the defaults are right for this setup. See
   [Settings reference](#settings-reference) for what you can change there.
5. **Deploy.** The first build takes a few minutes. The logs should end with
   `SillyTavern is listening on IPv4: 0.0.0.0:8000`.

Check it from the VPS. A `302` (redirect to the login page) means it is up:

```sh
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8001/
```

## 3. Publish it on your tailnet

On the VPS:

```sh
sudo tailscale serve --bg http://127.0.0.1:8001
tailscale serve status
```

`serve status` shows the address, e.g. `https://sillytavern.tail1234.ts.net`. Open it on any device
in your tailnet. The very first request can take a few seconds while the certificate is issued.

- `--bg` keeps it running across reboots. Turn it off with `sudo tailscale serve reset`.
- `serve` is tailnet-only. **Never use `tailscale funnel`** for this: that publishes it to the
  internet.
- If you changed `ST_LOCAL_PORT`, use that port instead of `8001`.
- Use `127.0.0.1`, not `localhost`, which may resolve to IPv6 `::1` where nothing listens.

## 4. Set a password

User accounts are on. The built-in admin account `default-user` starts **without a password**.
Anything on your tailnet could use it, so set one now.

1. Open the site. Discreet login hides the user list: type `default-user` as the handle and leave
   the password empty.
2. **User Settings** (top bar) → **Account** → **Change Password**.

Or from the Coolify **Terminal** tab of the `sillytavern` container:

```sh
su-exec node:node node recover.js default-user 'a-long-password'
```

Keep the `su-exec node:node` prefix. Running it as root can leave files the app cannot write.

For daily use you can create a separate non-admin user under **User Settings → Admin Panel**.

## 5. Lock down the VPS

Use the **Hetzner Cloud Firewall** (Console → Firewalls), not just `ufw`. It filters traffic before
it reaches the server, and Docker-published ports bypass `ufw` entirely.

Inbound rules:

| Port | Source | Why |
|---|---|---|
| TCP 80, 443 | Any | Only if Coolify serves public apps or its dashboard on a domain. Otherwise leave out. |
| TCP 22 | Your IP only, or nothing | Remove once SSH over Tailscale works (below). |
| UDP 41641 | Any | Optional. Lets Tailscale connect directly instead of through relays. |

Leave **8000, 6001 and 6002** (the Coolify dashboard) closed to the internet. Reach the dashboard
over Tailscale instead at `http://sillytavern:8000`. Tailnet traffic arrives inside Tailscale's
encrypted tunnel, so the Hetzner firewall does not block it. Leave outbound rules empty (allow all).

**SSH over Tailscale.** Your normal OpenSSH server already works over the tailnet. In a **second**
terminal, before touching the firewall, check:

```sh
ssh root@sillytavern        # or root@100.x.y.z
```

When that works, remove the public TCP 22 rule. If you ever lock yourself out, use the Hetzner
Console's browser console (VNC) or the Rescue system.

> **Coolify still needs port 22 internally.** Coolify manages its own server over SSH from a
> container to the host (`root@host.docker.internal`). The Hetzner firewall never sees that traffic,
> so closing 22 there is safe. If you use `ufw` too, allow 22 from the Docker networks
> (`sudo ufw allow from 10.0.0.0/8 to any port 22` and the same for `172.16.0.0/12`). Do not
> restrict `sshd` to the Tailscale IP only.

Also keep the OS patched, e.g. `sudo apt install unattended-upgrades`.

## 6. Optional: restrict who on the tailnet can reach it

If your tailnet has only your own devices, the default policy (everything can reach everything) is
fine. If you share it with others, tag the VPS and grant access only to yourself. In the admin
console:

1. **Access controls**: declare the tag and grant yourself access. A policy with grants is
   deny-by-default, so also keep access between your own devices:

   ```json
   {
     "tagOwners": { "tag:server": ["you@example.com"] },
     "grants": [
       { "src": ["you@example.com"], "dst": ["tag:server"], "ip": ["tcp:443", "tcp:22", "tcp:8000", "tcp:6001", "tcp:6002"] },
       { "src": ["autogroup:member"], "dst": ["autogroup:self"], "ip": ["*"] }
     ]
   }
   ```

   Port 443 is SillyTavern (via `serve`), 22 is SSH, and 8000/6001/6002 is the Coolify dashboard.
2. **Machines** → `sillytavern` → **⋯** → **Edit ACL tags** → add `tag:server`. Tagged machines
   have key expiry disabled by default.

Test SSH and the site from a second session after saving.

## Settings reference

Set these in Coolify's **Environment Variables** tab, then redeploy:

| Variable | Default | What it does |
|---|---|---|
| `SILLYTAVERN_WHITELISTMODE` | `true` | IP whitelist (tailnet addresses only). Set `false` only for the public-domain setup. |
| `SILLYTAVERN_ENABLEUSERACCOUNTS` | `true` | Login with user accounts. |
| `SILLYTAVERN_ENABLEDISCREETLOGIN` | `true` | Hide the list of users on the login page. |
| `SILLYTAVERN_SESSIONTIMEOUT` | `86400` | Seconds of inactivity before you are logged out. |
| `SILLYTAVERN_PRIVATEADDRESSWHITELIST_ENABLED` | `true` | Block server-side requests to private IPs (SSRF protection). |
| `ST_LOCAL_PORT` | `8001` | Loopback port on the VPS that `tailscale serve` points at. Not listed automatically; add it to change the port. |
| `PUID` / `PGID` | `1000` | User and group the app runs as. |

Booleans must be exactly `true` or `false`: SillyTavern reads `0`, `no` and `off` as **true**.

These are fixed in the compose file (edit the file to change them):

- **IP whitelist ranges.** Docker networks plus tailnet addresses (`100.64.0.0/10`,
  `fd7a:115c:a1e0::/48`). A request passes only if both the connecting IP (a Docker gateway when it
  comes from `tailscale serve`) and the client IP in `X-Forwarded-For` are listed.
- **Client IP headers.** Only `X-Forwarded-For` is trusted. `tailscale serve` and Coolify's Traefik
  both overwrite it. `tailscale serve` passes a client-supplied `X-Real-IP` through unchanged, so
  that header is off. Failed-login rate limiting uses the client IP, so one attacker cannot lock
  everyone out.
- **Host whitelist.** `.ts.net` (any Tailscale hostname); IPs and `localhost` are always allowed.
  Blocks DNS-rebinding attacks.
- **Outbound filter allowlist.** Loopback and tailnet IPv6. Tailnet IPv4 addresses are never
  blocked, so an LLM backend on another tailnet device works. If a backend runs in another Docker
  container or on a private network, add its range to
  `SILLYTAVERN_PRIVATEADDRESSWHITELIST_ALLOWEDRANGES`.

Variables set in Coolify cannot override the fixed values. Neither can `config.yaml` on the config
volume: it keeps showing the defaults, but the `SILLYTAVERN_*` variables always win.

## Public domain instead of Tailscale

Only do this if you need access without Tailscale. SillyTavern accounts have passwords but **no
two-factor login**, so put another login layer in front (step 5).

1. **Set a password first** ([step 4](#4-set-a-password)). With the whitelist off and a
   password-less admin, SillyTavern refuses to start and the container restarts in a loop.
2. **Allow the hostname.** In `docker-compose.coolify.yml`, add your domain to
   `SILLYTAVERN_HOSTWHITELIST_HOSTS`, e.g. `'[".ts.net","st.example.com"]'`. Commit and push.
3. **Add the domain** to the `sillytavern` service in Coolify as `https://st.example.com:8000` (the
   `:8000` is the container port; visitors still use 443).
4. **Turn the IP whitelist off:** set `SILLYTAVERN_WHITELISTMODE` to `false`. Redeploy.
5. **Add a second login layer**, for example Cloudflare Access, Authentik/Authelia (SillyTavern
   supports both through `sso.*` in `config.yaml`), or Traefik basic auth through Coolify's
   `coolify.traefik.middlewares` label.

The loopback port stays published. It is harmless and keeps Tailscale access working. After each
start the domain answers with "no available server" for about 30 seconds, until the healthcheck
passes. If the domain is proxied through Cloudflare, SillyTavern sees Cloudflare's IPs as clients
unless you configure trusted IPs in Coolify's proxy.

## Updating

- Push to the deployed branch and Coolify redeploys (automatic with the GitHub App, or click
  **Redeploy**). Each redeploy stops SillyTavern for a moment: Coolify has no rolling updates for
  compose apps.
- To pull in upstream changes:
  `git remote add upstream https://github.com/SillyTavern/SillyTavern.git` (once), then
  `git fetch upstream && git merge upstream/release` and push.

## Backups

All data lives in four Docker volumes (`<resource-uuid>_st-data`, `_st-config`, `_st-plugins`,
`_st-extensions`).

- The data volume holds chats, characters **and every user's API keys in plaintext**
  (`secrets.json`). Store backups as carefully as passwords.
- **Coolify:** Persistent Storage → **Configure Backup** on each volume. Enable **Stop containers
  during archive** for consistent copies. There is no restore button; restore by extracting the
  archive into the volume.
- **Hetzner:** server Backups or snapshots also cover the volumes.
- **Deleting the resource** in Coolify, with the default boxes ticked, deletes these volumes. Untick
  them or back up first.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `403` page "Forbidden by whitelist" | The client IP is not a tailnet address. The container log shows `Blocked connection from X (forwarded from Y)`. Expected for requests through a Coolify domain while `SILLYTAVERN_WHITELISTMODE=true`. |
| `403` page about the host | The hostname is not in `SILLYTAVERN_HOSTWHITELIST_HOSTS`. Entries are exact and lowercase; a leading dot matches subdomains. |
| Container keeps restarting, log says "you should set a password for all admin users" | Whitelist off while an admin has no password. Set `SILLYTAVERN_WHITELISTMODE=true`, redeploy, set the password ([step 4](#4-set-a-password)), then switch back. |
| `tailscale serve` shows 502 | SillyTavern is not running or the port differs. Check `curl -sI http://127.0.0.1:8001/` on the VPS. |
| Build fails or gets killed | Out of memory during webpack. Add swap or use a larger server. |
| An LLM backend in another container is unreachable, log shows `[Private Request Filter]` | Add its network (e.g. `10.0.0.0/8`) to `SILLYTAVERN_PRIVATEADDRESSWHITELIST_ALLOWEDRANGES` in the compose file. |
| Edits to `config.yaml` have no effect | The compose file sets that option through a `SILLYTAVERN_*` variable, which always wins. |
| Never set `SILLYTAVERN_HOSTWHITELIST_ENABLED=false` | SillyTavern reads any non-empty value as true. Remove the line from the compose file instead. |
