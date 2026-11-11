# Media stack (private, VPN-only)

A hardlink-friendly \*arr + Jellyfin + ebook stack, all in the `media`
namespace on the single k3s node. Reach the WebUIs over the Headscale mesh or
LAN:

| App           | URL over LAN/mesh                                 | NodePort |
|---------------|---------------------------------------------------|----------|
| Jellyfin      | http://<node-ip>:30096                            | 30096    |
| qBittorrent   | http://<node-ip>:30080                            | 30080    |
| Prowlarr      | http://<node-ip>:30696                            | 30696    |
| Sonarr        | http://<node-ip>:30989                            | 30989    |
| Radarr        | http://<node-ip>:30787                            | 30787    |
| Calibre-Web   | http://calibre.home.barnabash.com or :30883       | 30883    |
| LazyLibrarian | http://lazylibrarian.home.barnabash.com or :30529 | 30529    |

The Calibre content server is internal-only at
`http://calibre-content-server:8081`; it has no Ingress or NodePort. Bonjour
advertising is disabled because Kubernetes clients use the Service DNS name.

## Storage layout (single filesystem = instant hardlinks)

```
/mnt/hdd/data/
  torrents/            qBittorrent save path (categories: tv, movies, books)
  media/
    movies/            Radarr root folder  -> Jellyfin library
    tv/                Sonarr root folder  -> Jellyfin library
    books/             Calibre-Web library (contains metadata.db)
```

- `media-data` PVC (hostPath `/mnt/hdd/data`) is mounted at `/data` in
  qBittorrent, Sonarr and Radarr — so imports hardlink (no copy, no extra space)
  and seeding continues from the same bytes.
- `jellyfin-media` PVC (hostPath `/mnt/hdd/data/media`) is mounted at `/media`
  in Jellyfin only (it never sees `torrents/`).
- `calibre-library` PVC is mounted at `/books` in Calibre-Web,
  LazyLibrarian, and the Calibre content server.
- LazyLibrarian mounts the same library at `/books` and the shared data volume
  at `/data`; book downloads can stage under `/data/torrents/books`.
- All apps run as UID/GID **1000** so ownership is consistent.

## One-time host prep

Mount the data disk at `/mnt/hdd` on the node before starting workloads, then
create the directories and set ownership:

```sh
sudo mkdir -p /mnt/hdd/data/torrents/{tv,movies,books,legacy} /mnt/hdd/data/media/{movies,tv,books}
# Seed an empty Calibre database only when this is a new library.
if [ ! -f /mnt/hdd/data/media/books/metadata.db ]; then
  sudo curl -fL https://raw.githubusercontent.com/janeczku/calibre-web/master/library/metadata.db \
    -o /mnt/hdd/data/media/books/metadata.db
fi
# migrate existing downloads into the tree (instant: same filesystem)
sudo mv /mnt/hdd/Downloads /mnt/hdd/data/torrents/legacy
# make everything owned by the shared UID/GID
sudo chown -R 1000:1000 /mnt/hdd/data
```

## Wiring (in the WebUIs, after deploy)

1. **qBittorrent**: default save path `/data/torrents`; make categories `tv`,
   `movies`, and `books` (save to their matching subdirectories). Grab the
   WebUI temp password from logs:
   `kubectl -n media logs deploy/qbittorrent | grep -i password`.
2. **Prowlarr**: add indexers; add Sonarr, Radarr, and LazyLibrarian under
   **Settings → Apps** so their indexers sync. For LazyLibrarian, use
   `http://lazylibrarian:5299` as the application URL and its API key from
   Byparr is a FlareSolverr-compatible **indexer proxy**, not an indexer:
   configure it under **Settings → Indexers → Indexer Proxies → FlareSolverr**
   with host `http://byparr:8191`, then test it. Give the proxy a tag such as
   `byparr` and apply that tag only to indexers that need Cloudflare solving.
   Leave other indexers untagged so they connect directly.
3. **Sonarr**: root folder `/data/media/tv`; add qBittorrent as download client
   (host `qbittorrent`, port `8080`), category `tv`. Enable "Use Hardlinks".
4. **Radarr**: root folder `/data/media/movies`; download client `qbittorrent:8080`,
   category `movies`. Enable "Use Hardlinks".
5. **Calibre-Web**: browse to `http://calibre.home.barnabash.com` or
   `http://<node-ip>:30883` over LAN/Headscale. Sign in with `admin` /
   `admin123`, immediately change the password, and set the Calibre database
   location to `/books`. Add books manually using Calibre-Web's upload option.
   The OPDS catalog is at `/opds`; an iPad reader can connect to
   `http://<node-ip>:30883/opds`.
6. **Calibre content server**: it runs internally with authentication and
   write access. Get the LazyLibrarian service password with
   `sops -d apps/base/media/calibre-content-server-secret.sops.yaml`; the
   username is `lazylibrarian`.
7. **LazyLibrarian**: open `http://lazylibrarian.home.barnabash.com/home` or
   `http://<node-ip>:30529/home` over LAN/Headscale and enable access control
   with your own credentials. Set the book library destination to `/books`.
   Under **Settings → Processing → Calibre**, set the content-server URL to
   `http://calibre-content-server:8081`, then enter username `lazylibrarian`
   and the password from the SOPS secret. Leave the `calibredb` import program
   and Calibre auto-add directory unset; the authenticated content server
   handles library writes.
   For a manual test, set the Alternate Import Folder to
   `/data/torrents/books`, import a test ebook you own, and verify it appears in
   Calibre-Web before enabling scheduled searches or downloaders. If you later
   enable downloads, use qBittorrent at `qbittorrent:8080` and stage them in
   `/data/torrents/books`.
8. Calibre-Web and LazyLibrarian both update the same Calibre database; avoid
   uploading books in Calibre-Web while LazyLibrarian is importing.
9. **Jellyfin**: add libraries pointing at `/media/movies` and `/media/tv`.
10. **Import legacy**: in Radarr/Sonarr use *Library Import* / *Manual Import*
   against `/data/torrents/legacy` — files hardlink into `/data/media/*`.

## Jellyfin 12 upgrade

The Jellyfin chart now defaults to server 12.4. Before its first start, back up
the `jellyfin-config` PVC and check that no usernames differ only by letter
case. Remove third-party plugins built for 10.11; they will not load on 12.
Do not interrupt the first startup while the database migrates, and run a full
library scan afterward. Older clients that depend on `/emby/` or
`/mediabrowser/` endpoints may no longer work.

The upstream `jellyfin/jellyfin` and LinuxServer images use different layouts
inside `/config`: upstream stores data at `/config` and config under
`/config/config`, while LinuxServer expects config at `/config` and data under
`/config/data`. Back up and migrate the PVC contents with Jellyfin stopped
before switching between these images. The deployment uses a `Recreate`
strategy so two pods cannot write to the same SQLite databases during an update.

## Prowlarr and Radarr upgrades

Back up the `prowlarr-config` and `radarr-config` PVCs before reconciling these
image updates; both apps will migrate their databases on startup.

## Seerr migration

Seerr reuses the `jellyseerr-config` PVC mounted at `/app/config` and migrates
the Jellyseerr instance automatically on first startup. Back up that PVC before
the image upgrade.

Do not port-forward these services from the public internet.

## Optional: route torrent traffic through a VPN provider (later)

Add a Gluetun sidecar to the qBittorrent controller (CyberGhost VPN),
force qBittorrent's network through it with a kill-switch, and put the provider
creds in a `*.sops.yaml` secret. Skipped for now to keep full 1 Gbps throughput.
