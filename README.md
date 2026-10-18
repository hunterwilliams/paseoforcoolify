# paseo-for-coolify

Run [Paseo](https://paseo.sh) on [Coolify](https://coolify.io) and drive coding agents from your phone, desktop or browser, over your private [Tailscale](https://tailscale.com) network.

A Tailscale container joins your tailnet and proxies HTTPS to the official Paseo image over the stack's internal Docker network. Paseo is reachable only from your own devices at `https://paseo.<tailnet>.ts.net`. Nothing is published on the server: no ports, no public domain, no reverse proxy. The agent used here is [Pi](https://pi.dev), but any agent Paseo supports works the same way.

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Tailscale sidecar and Paseo |
| `.env.example` | Variables for running outside Coolify |

## Before you deploy

1. **Enable HTTPS on your tailnet.** In the [Tailscale admin console](https://login.tailscale.com/admin/dns), turn on **MagicDNS**, then under **HTTPS Certificates** click **Enable HTTPS**.
2. **Create an auth key.** Go to [Settings → Keys](https://login.tailscale.com/admin/settings/keys) and click **Generate auth key**. Turn **Reusable** on and leave **Ephemeral** off. Copy the key.
3. **Install Tailscale** on each device you'll connect from, signed in to the same tailnet.

Tailscale's free Personal plan is for non-commercial use. For client work, use a paid plan or Headscale.

## Deploy on Coolify

1. In Coolify, open your project and click **+ New**.
2. Choose **Public Repository** and paste this repo's URL. For a private fork, choose **Private Repository (with GitHub App)** instead.
3. Set **Build Pack** to **Docker Compose**, then click **Continue**.
4. Open the **Environment Variables** tab:
   - `TS_AUTHKEY`: paste the auth key. Deployment is blocked until this is set.
   - `SERVICE_PASSWORD_PASEO`: generated automatically. This is the daemon password; copy it for later.
   - Set `TZ` (for example `Asia/Bangkok`), and `GIT_NAME` and `GIT_EMAIL` for commits made by agents. These are written to the global git config on every start by the short-lived `git-config` container, which then shows as exited in Coolify. That is expected.
   - Fill in the API key for the provider you use (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY` or `OPENROUTER_API_KEY`). Leave the rest empty.
5. Leave **Domains** empty for both services.
6. Click **Deploy**.

The server should appear as `paseo` in the [Tailscale machines list](https://login.tailscale.com/admin/machines).

## Install Pi

Open the **Terminal** tab in Coolify and select the `paseo` container. The terminal runs as root, so the command switches to the `paseo` user and carries over the container's `PATH`:

```sh
su paseo -s /bin/sh -c "PATH='$PATH' npm install -g --prefix /home/paseo/.local @earendil-works/pi-coding-agent"
```

Then click **Restart** in Coolify so Paseo detects Pi.

## Connect

Your address is `https://paseo.<tailnet>.ts.net`. Find the exact name in the Tailscale machines list.

- **Browser:** open the address and enter the daemon password.
- **iPhone, Android, Mac, Windows, Linux:** install Paseo from the App Store, Google Play or [paseo.sh/download](https://paseo.sh/download). Go to **Settings → Add host → Direct connection** and enter:
  - **Host:** `paseo.<tailnet>.ts.net`
  - **Port:** `443`
  - **Use SSL:** on
  - **Password:** the daemon password

Tailscale must be connected on the device. On iOS, turn on **VPN On Demand** in the Tailscale app so it connects automatically.

## Using a subscription instead of an API key

Run `pi` once in the terminal and use `/login`:

```sh
su paseo -s /bin/sh -c "cd /workspace && PATH='$PATH' pi"
```

The credential is saved in the `paseo-home` volume. Check your provider's terms before using a consumer subscription through a third-party agent.

## GitHub access

Paseo uses the GitHub CLI (`gh`) for cloning and pull requests. The image doesn't include it.

**Install `gh`.** In the `paseo` container terminal, as root (no `su`):

```sh
apt-get update && apt-get install -y gh
```

This installs into the container itself, not a volume, so it is lost when the container is recreated (a redeploy or image update). Rerun it after each redeploy.

**Use a dedicated GitHub account.** Agents can read any credential in the container, so don't use your personal account. Create a separate account, invite it with **Write** access to each repo it should work on, then create a classic token on it with the `repo` scope (**Settings → Developer settings → Personal access tokens → Tokens (classic)**). One account works across repos with different owners; a fine-grained token can't. If an organization uses SAML single sign-on, authorize the token for it under **Configure SSO**.

**Log in and switch git to HTTPS**, as the `paseo` user:

```sh
su paseo -s /bin/sh -c 'gh auth login --with-token && gh auth setup-git'
```

Paste the token, press **Enter**, then **Ctrl+D**. Then make every GitHub address use HTTPS, so clones use the token instead of a missing SSH key:

```sh
su paseo -s /bin/sh -c 'git config --global url."https://github.com/".insteadOf "git@github.com:" && gh config set git_protocol https'
```

The login and git settings are stored in the `paseo-home` volume and survive redeploys.

**If `gh` reports `permission denied` on its config**, a command was run as root and created files in the `paseo` home folder. Fix ownership with `chown -R 1000:1000 /home/paseo`, then repeat the login.

## Repositories

Add repositories from the Paseo app. It clones them into `/home/paseo/dev`, inside the persistent `paseo-home` volume.

## Updating

Change `PASEO_VERSION` under **Environment Variables** and redeploy. Image tags are `X.Y.Z` (no `v`); see the [changelog](https://paseo.sh/changelog). To update Pi, rerun the install command.

## Swapping the network layer

Paseo doesn't depend on Tailscale. The `tailscale` service is just a proxy that reaches Paseo at `paseo:6767` on the stack's internal network. To use something else (Netbird, a Cloudflare Tunnel, a reverse proxy), replace that service with one that forwards to `paseo:6767`, or remove it and use the relay below.

To share one Tailscale node across several Coolify apps instead, deploy it as its own resource and turn on **Connect To Predefined Network** (under the resource's **Advanced** settings) for it and for this stack, so they share the `coolify` network. Coolify may prefix service names on that network, so check the generated name before pointing the proxy at it.

## Headscale

Set `TS_EXTRA_ARGS` to `--login-server=https://headscale.example.com` and use a Headscale pre-auth key as `TS_AUTHKEY`. Headscale doesn't support Tailscale Serve, so the proxy won't work. Instead, add `network_mode: service:tailscale` to the `paseo` service so it runs inside the Tailscale container's network, then connect with **Host** `paseo.<your base domain>`, **Port** `6767` and **Use SSL** off. Traffic is still encrypted by WireGuard; the web UI's microphone and clipboard features need HTTPS and won't work in the browser.

## Optional: relay

Set `PASEO_RELAY_ENABLED=true` to also allow pairing through Paseo's hosted, end-to-end encrypted relay, for example from a device without Tailscale. Pair from the terminal:

```sh
su paseo -s /bin/sh -c "PATH='$PATH' PASEO_PASSWORD='$PASEO_PASSWORD' paseo daemon pair --relay"
```

The pairing link is a credential. The relay is free under fair use with no uptime guarantee, and it sees connection metadata.

## Security notes

- **Pi has no sandbox.** Its tools run with the full permissions of the process. The container is the only boundary, so agents can read everything in the volumes, including API keys and logins.
- **Use dedicated API keys with a spending limit.** Any environment variable is visible to the agents and everything they run.
- **Push notifications** to the mobile app likely pass through a third-party push service. This is undocumented by Paseo; disable notifications on your phone if that matters.
- **Don't mount the Docker socket** into this container. It gives agents root on the host.
- **Back up the volumes** if they hold unpushed work. Push branches often.

## Running without Coolify

```sh
cp .env.example .env   # set TS_AUTHKEY, SERVICE_PASSWORD_PASEO and an API key
docker compose up -d
docker compose exec --user paseo paseo npm install -g --prefix /home/paseo/.local @earendil-works/pi-coding-agent
docker compose restart paseo
```

## License

MIT
