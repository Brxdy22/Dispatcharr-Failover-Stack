# Dispatcharr Failover Stack

How to install three projects side by side for [Dispatcharr](https://github.com/Dispatcharr/Dispatcharr):

| Project | Type | What it does |
|---|---|---|
| [M3U Failover](https://github.com/Brxdy22/Dispatcharr-M3U-Failover) | Docker container | Gives an M3U account several provider URLs and fails over between them |
| [VOD Failover](https://github.com/Brxdy22/Dispatcharr-VOD-Failover-Plugin) | Dispatcharr plugin | Retries a failed VOD stream on the next account that carries the title |
| [Gluetun AutoSwitch](https://github.com/Brxdy22/Dispatcharr-Gluetun-AutoSwitch-Plugin) | Dispatcharr plugin | Moves [Gluetun](https://github.com/qdm12/gluetun) to another VPN exit when streams slow down or a provider refuses the exit |

Together they cover a dead provider URL, a provider account that can't play a title, and a VPN exit that is slow or refused.

This repo holds only instructions and examples. Each project is installed from its own repo.

## Tested together

| Dispatcharr | M3U Failover | VOD Failover | Gluetun AutoSwitch |
|---|---|---|---|
| 0.33.0 | 1.3.1 | 1.5 | 1.8.4.1 |

## What needs what

Each project works alone, but some features need another one:

| Feature | Needs |
|---|---|
| Mirror URLs section in VOD Failover | M3U Failover 1.3 or later |
| VOD rejection switching in AutoSwitch | VOD Failover enabled |
| Switch at the first refused VOD, background VOD check | M3U Failover 1.2 or later in front of the account |
| Closing streams just before a switch | M3U Failover 1.2.1 or later |
| Faster reconnect after a switch | M3U Failover 1.1.0 or later |
| Any exit switching | Dispatcharr running through Gluetun, on Mullvad |

## How it fits

```
Player ──> Dispatcharr ──> M3U Failover (127.0.0.1:<port>) ──> provider URL 1, 2, 3 ...
              │  plugins:                  ▲
              │   VOD Failover ────────────┘ edits URL lists, retries other accounts
              │   Gluetun AutoSwitch ──> Gluetun control server (127.0.0.1:8000)
              │          ▲
              │          └── M3U Failover asks it to move a refused exit (127.0.0.1:9733)
              └── all of this runs inside Gluetun's network, so provider traffic uses the VPN
```

## Install

Do these in order. The example files in this repo use placeholders in `<angle brackets>`.

### 1. Gluetun and Dispatcharr

Use [`docker-compose.example.yml`](docker-compose.example.yml) as a base. The important parts:

- Dispatcharr and M3U Failover use `network_mode: "service:gluetun"`.
- Publish Dispatcharr's port (9191) on the **gluetun** service.
- Mount Gluetun's `servers` folder read-only at `/gluetun-servers` in Dispatcharr, for AutoSwitch's country and city lists.
- Keep `FIREWALL_OUTBOUND_SUBNETS` to your LAN. A wide range such as `172.0.0.0/8` also covers public Cloudflare addresses, so provider traffic leaves outside the VPN.

### 2. M3U Failover

1. Copy [`upstreams.conf.example`](https://github.com/Brxdy22/Dispatcharr-M3U-Failover/blob/main/upstreams.conf.example) to `./m3u-failover/upstreams.conf`. Add one section per M3U account, with a port and all of that provider's URLs.
2. Start it: `docker compose up -d m3u-failover`.
3. In Dispatcharr, set each account's **Server URL** to `http://127.0.0.1:<port>`, then refresh the account. Usernames and passwords stay in Dispatcharr.

Channel and VOD assignments are kept, because Xtream Codes streams are matched by stream ID, not by URL.

### 3. VOD Failover

1. Download the plugin zip from the [latest release](https://github.com/Brxdy22/Dispatcharr-VOD-Failover-Plugin/releases/latest) and import it on Dispatcharr's Plugins page. Use the release zip, not GitHub's source zip.
2. Enable it. It needs at least two accounts carrying the same titles.
3. Leave **M3U Failover address** at `127.0.0.1`.
4. Press **Check Hook Status**, then **Mirror Status**. Every account pointed at M3U Failover should be listed.

### 4. Gluetun AutoSwitch

1. Save [`gluetun-auth.example.toml`](gluetun-auth.example.toml) as `./gluetun/auth/config.toml` with your own long random key, then restart Gluetun.
2. Download the plugin zip from the [latest release](https://github.com/Brxdy22/Dispatcharr-Gluetun-AutoSwitch-Plugin/releases/latest) and import it.
3. Set **Gluetun URL** to `http://127.0.0.1:8000` and **Gluetun API key** to the key from step 1.
4. Pick at least one **Allowed country**, save, and refresh the page to fill the city list.
5. Press **Check status**, then turn on **Auto switch**.

Restarting Gluetun recreates its network, so restart Dispatcharr and M3U Failover after it: `docker compose up -d`.

## Check it all works

```sh
# M3U Failover: accounts, URLs, current streams
docker exec m3u-failover python /app/failover.py --status
```

- VOD Failover: **Check Hook Status** says the hook is active.
- Gluetun AutoSwitch: **Check status** reaches Gluetun and shows the current exit, and the VOD hook is listening.

## Updating

- Update **M3U Failover first**, then the plugins. Newer plugins call proxy routes that older proxies would forward to your provider.
- M3U Failover: change the image tag and run `docker compose up -d --no-deps m3u-failover`.
- Plugins: import the new release zip on the Plugins page. Check the hook status after any Dispatcharr update, because VOD Failover patches Dispatcharr's internal VOD functions.
- Use `--no-deps` when redeploying one service. Without it, Compose may recreate Gluetun too, which drops every stream.

## Security

- Keep `upstreams.conf`, the Gluetun auth file and your WireGuard key out of Git.
- M3U Failover has no login, so keep it on `127.0.0.1`.
- Don't publish Gluetun's control server port unless you need it.

## Licence

MIT
