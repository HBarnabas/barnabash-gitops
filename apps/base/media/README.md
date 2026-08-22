# Media server

Defaults to **Jellyfin** (fully FOSS, no account/license needed). Swap to Plex
or Emby if you prefer:

## Using Plex instead

Replace `helmrelease.yaml` with a Plex chart, e.g. the `plex-media-server`
chart (`https://raw.githubusercontent.com/plexinc/pms-docker/gh-pages`) or the
bjw-s `app-template` running `ghcr.io/linuxserver/plex`. Plex needs a claim
token on first run (`PLEX_CLAIM`) — store it in a `*.sops.yaml` secret.

Keep the same `media-library` PVC so the rest of the repo is unchanged.

## Sharing the library with the download client

PVCs are namespace-scoped and `local-path` is ReadWriteOnce, so the `downloads`
namespace cannot mount this exact PVC. Options, best first:

1. **RWX storage (recommended):** back the library with NFS (or Longhorn RWX)
   and reference the same share from both `media` and `downloads`.
2. **Single namespace:** move qBittorrent into the `media` namespace and mount
   `media-library` directly.
3. **Separate volumes + move step:** let qBittorrent download into its own PVC
   and periodically move/hardlink into the library (e.g. via *arr apps).

## Hardware transcoding

Only enable on nodes with a suitable GPU/VAAPI (i.e. the PC, not the Pi). Mount
`/dev/dri` and add the `render`/`video` supplemental groups via the pc-k8s
overlay.
