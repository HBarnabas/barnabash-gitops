# Media stack (private, VPN-only)

A hardlink-friendly \*arr + Jellyfin stack, all in the `media` namespace on the
single Pi node. Reach each WebUI over the WireGuard tunnel (NodePorts):

| App         | URL over VPN                | NodePort |
|-------------|-----------------------------|----------|
| Jellyfin    | http://<pi-ip>:30096        | 30096    |
| qBittorrent | http://<pi-ip>:30080        | 30080    |
| Prowlarr    | http://<pi-ip>:30696        | 30696    |
| Sonarr      | http://<pi-ip>:30989        | 30989    |
| Radarr      | http://<pi-ip>:30787        | 30787    |

## Storage layout (single filesystem = instant hardlinks)

```
/mnt/hdd/data/
  torrents/            qBittorrent save path (categories: tv, movies)
  media/
    movies/            Radarr root folder  -> Jellyfin library
    tv/                Sonarr root folder  -> Jellyfin library
```

- `media-data` PVC (hostPath `/mnt/hdd/data`) is mounted at `/data` in
  qBittorrent, Sonarr and Radarr — so imports hardlink (no copy, no extra space)
  and seeding continues from the same bytes.
- `jellyfin-media` PVC (hostPath `/mnt/hdd/data/media`) is mounted at `/media`
  in Jellyfin only (it never sees `torrents/`).
- All apps run as UID/GID **1000** so ownership is consistent.

## One-time host prep

```sh
sudo mkdir -p /mnt/hdd/data/torrents /mnt/hdd/data/media/movies /mnt/hdd/data/media/tv
# migrate existing downloads into the tree (instant: same filesystem)
sudo mv /mnt/hdd/Downloads /mnt/hdd/data/torrents/legacy
# make everything owned by the shared UID/GID
sudo chown -R 1000:1000 /mnt/hdd/data
```

## Wiring (in the WebUIs, after deploy)

1. **qBittorrent**: default save path `/data/torrents`; make categories `tv` and
   `movies` (save to `/data/torrents/tv` and `/data/torrents/movies`). Grab the
   WebUI temp password from logs: `kubectl -n media logs deploy/qbittorrent | grep -i password`.
2. **Prowlarr**: add indexers; add Sonarr + Radarr as apps (sync indexers).
3. **Sonarr**: root folder `/data/media/tv`; add qBittorrent as download client
   (host `qbittorrent`, port `8080`), category `tv`. Enable "Use Hardlinks".
4. **Radarr**: root folder `/data/media/movies`; download client `qbittorrent:8080`,
   category `movies`. Enable "Use Hardlinks".
5. **Jellyfin**: add libraries pointing at `/media/movies` and `/media/tv`.
6. **Import legacy**: in Radarr/Sonarr use *Library Import* / *Manual Import*
   against `/data/torrents/legacy` — files hardlink into `/data/media/*`.

## Optional: route torrent traffic through a VPN provider (later)

Add a Gluetun sidecar to the qBittorrent controller (CyberGhost via WireGuard),
force qBittorrent's network through it with a kill-switch, and put the provider
creds in a `*.sops.yaml` secret. Skipped for now to keep full 1 Gbps throughput.
