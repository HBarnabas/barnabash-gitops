# Download client

Defaults to **qBittorrent** (via the bjw-s `app-template` chart). Undecided app,
so swapping is easy — just change the container image and ports:

- **Transmission:** `ghcr.io/home-operations/transmission`
- **Deluge:** `ghcr.io/home-operations/deluge`
- **SABnzbd (usenet):** `ghcr.io/home-operations/sabnzbd`

## Strongly recommended: route traffic through a VPN

For a public torrent client, add a VPN sidecar (e.g. Gluetun) in the same pod
and force qBittorrent's traffic through it, with a kill-switch. Put the VPN
credentials in a `*.sops.yaml` secret. The bjw-s `app-template` supports adding
a second container for exactly this.

## Getting files into the media library

See `apps/base/media/README.md` — the cleanest option is an NFS/RWX share
mounted by both this app and the media server.

## Default login

The linuxserver/home-operations qBittorrent image logs a temporary WebUI
password to the container logs on first start:

```sh
kubectl -n downloads logs deploy/qbittorrent | grep -i password
```
