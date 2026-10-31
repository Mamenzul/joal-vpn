# JOAL behind Proton VPN (Docker)

Two containers:

| service | image | role |
|---|---|---|
| `vpn` | `qmcgaw/gluetun:v3.41.3` | Proton VPN WireGuard tunnel (gluetun's built-in `protonvpn` provider), firewall/kill switch, owns the network namespace |
| `joal` | `anthonyraymond/joal:2.1.37` | ratio daemon + WebUI, running **inside** the VPN namespace (`network_mode: service:vpn`) |

Everything you configure lives in one file, `.env`. Nothing else holds secrets.

## Prerequisites

- **Windows**: [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/) with the WSL 2 backend (the default).
- **macOS**: [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) (Intel or Apple Silicon).
- **Linux**: Docker Engine + the compose plugin. (Podman also works: replace `docker` with `podman`.)
- A Proton VPN account (any paid plan with WireGuard).

## Deployment

1. **Clone the repo**

   ```sh
   git clone https://github.com/Mamenzul/joal-vpn.git
   cd joal-vpn
   ```

2. **Create your settings file**

   ```sh
   cp .env.example .env              # macOS / Linux
   Copy-Item .env.example .env       # Windows PowerShell
   ```

3. **Get your Proton WireGuard private key.** At <https://account.protonvpn.com> → *Downloads* →
   *WireGuard configuration*, create a config (any platform, any server), and copy the value of its
   `PrivateKey = ...` line. Put that value in `.env` as `WIREGUARD_PRIVATE_KEY`, then throw the file
   away. gluetun picks the server itself.

4. **Fill in the rest of `.env`**
   - `SERVER_COUNTRIES`: one or more Proton countries, comma-separated (e.g. `Switzerland`).
     If you want, you can also add `- SERVER_CITIES=${SERVER_CITIES}` to `docker-compose.yml` and set it in `.env`.
   - `JOAL_PATH_PREFIX` and `JOAL_SECRET_TOKEN`: long random strings. They are the WebUI's URL path
     and password. To generate one:
     `openssl rand -hex 16` (macOS/Linux) or
     `-join ((48..57)+(97..122) | Get-Random -Count 32 | % {[char]$_})` (PowerShell).
   - `LAN_SUBNET`: your home network, e.g. `192.168.1.0/24`.
   - `TZ`, `JOAL_PORT`: defaults are fine.

5. **Start the stack**

   ```sh
   docker compose up -d
   docker compose logs -f vpn        # wait for "Public IP address is ..." then Ctrl+C
   ```

6. **Open the WebUI:** `http://127.0.0.1:8080/<JOAL_PATH_PREFIX>/ui/`. Enter the address `127.0.0.1`,
   port `8080`, path prefix and secret token from `.env` in the connection screen.

7. **Add torrents** by dropping `.torrent` files into `joal-data/torrents/`, then press start in the UI.
   You don't need to restart anything: JOAL watches the folder and loads new files within a second.

Day-to-day:

```sh
docker compose up -d       # start / apply .env changes
docker compose down        # stop
docker compose pull && docker compose up -d   # after bumping image tags
```

**DNS note:** Proton blocks Cloudflare, both plain `1.1.1.1` and DNS-over-TLS on :853, by resetting
the connections. gluetun uses Cloudflare DoT by default, which leaves the tunnel routing fine but
resolving nothing. So the compose file sets `DOT=off` + `DNS_ADDRESS=9.9.9.9`. That is plain Quad9,
queried *inside* the tunnel, so only Proton can see those queries, and Proton already carries all
your traffic.

## Client emulation

JOAL announces itself as **qBittorrent v5.2.3 (64-bit)** via `joal-data/clients/qbittorrent-5.2.3.client`
(peer-id `-qB5230-…`, `User-Agent: qBittorrent/5.2.3`). Upstream regenerates these files weekly,
so refresh yours when qBittorrent ships a new release:

```sh
curl -fsSL -o joal-data/clients/qbittorrent-<ver>.client \
  https://raw.githubusercontent.com/anthonyraymond/joal/master/resources/clients/qbittorrent-<ver>.client
# then update "client" in joal-data/config.json and: docker compose restart joal
```

## Tuning (`joal-data/config.json`)

```json
{
  "minUploadRate": 500, "maxUploadRate": 2000, "simultaneousSeed": 100,
  "client": "qbittorrent-5.2.3.client",
  "keepTorrentWithZeroLeechers": false, "uploadRatioTarget": 3.0
}
```

* `min/maxUploadRate` is **one global kB/s budget**, not a per-torrent speed. JOAL re-rolls it every
  20 minutes and splits it across the active torrents by leecher count. Lower values (e.g. 50–150)
  look more like a residential line.
* `simultaneousSeed` sets how many torrents announce at once. Fewer torrents give each one a more
  plausible share of the budget and avoid announcing dozens of swarms from one peer-id.
* `keepTorrentWithZeroLeechers: false` drops swarms that have no leechers instead of announcing
  them forever at 0 kB/s, which no real client does.
* `uploadRatioTarget: 3.0` retires a torrent once it reaches 3× its size, so session ratios stay
  possible.

What this does **not** hide: JOAL transfers no real data. A tracker that compares your claimed upload
with what peers report downloading will still see the mismatch. Also, without a forwarded port
through the VPN, the announced port accepts no incoming connections.

## Troubleshooting

1. **`vpn` keeps restarting / no handshake**: check `WIREGUARD_PRIVATE_KEY` (copy the whole value,
   no quotes or spaces) and that the key hasn't been revoked in your Proton account. Run
   `docker compose logs vpn`.
2. **WebUI unreachable from another machine**: change the port mapping to `"0.0.0.0:${JOAL_PORT}:${JOAL_PORT}"`
   and check that `LAN_SUBNET` matches your network.
3. **`joal` has no network after the VPN container was recreated** (not just restarted): its
   namespace no longer exists. Run `docker compose up -d --force-recreate joal`.
4. **Nothing resolves / announces time out**: check DNS first, then the tunnel:
   `docker exec joal-vpn nslookup tracker.opentrackr.org`. If that fails, try a different `DNS_ADDRESS`
   (`8.8.8.8` also works with Proton; `1.1.1.1` does not).
5. **Linux with SELinux only** (Fedora/RHEL). If you see `Failed to read file [...] AccessDeniedException`
   for a torrent, the file kept a host label: run `chcon -R -t container_file_t joal-data/` and then
   `touch joal-data/torrents/*.torrent`. Keep the `:Z` suffix on the mount; Docker Desktop ignores it.

## Verify the tunnel

Wait ~20 s after `up -d` before testing, because DNS isn't ready right away.

```sh
# exit IP must be Proton's, not your ISP's (compare with https://ipinfo.io/ip in a browser)
docker exec joal-vpn wget -qO- https://ipinfo.io/ip

# joal really lives in gluetun's namespace
docker exec joal grep -c tun0 /proc/net/dev                  # -> 1

# the IP joal reports to trackers
docker logs joal 2>&1 | grep 'IP reported to tracker'

# kill switch: with the VPN stopped, joal must lose all connectivity
docker exec joal bash -c 'timeout 6 bash -c "exec 3<>/dev/tcp/34.117.59.81/443" && echo REACHABLE || echo BLOCKED'
docker stop joal-vpn
docker exec joal bash -c 'timeout 6 bash -c "exec 3<>/dev/tcp/34.117.59.81/443" && echo REACHABLE || echo BLOCKED'
docker compose up -d --force-recreate
```
