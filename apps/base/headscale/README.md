# Headscale

Headscale is the self-hosted control server for the private mesh. It is
available at `https://vpn.barnabash.com`; its embedded DERP relay uses that
HTTPS endpoint and UDP port 3478 for STUN.

## Network requirements

- The existing public `vpn.barnabash.com` DDNS record must point to the server.
- Forward TCP 80 and 443 on the router to the k3s ingress-nginx service so
  cert-manager can issue the TLS certificate and remote clients can reach
  Headscale.
- Forward UDP 3478 to the k3s node for the embedded DERP server's STUN service.
- Allow TCP 80/443 and UDP 3478 through the NixOS host firewall.
- Pi-hole resolves `vpn.barnabash.com` to the server's LAN address for clients
  already on the home network.

For example, the NixOS firewall configuration needs:

```nix
networking.firewall.allowedTCPPorts = [ 80 443 ];
networking.firewall.allowedUDPPorts = [ 3478 ];
```

The Headscale DERP configuration disables Tailscale's public DERP map. The
embedded relay therefore requires the UDP 3478 forward; direct peer connections
can still work without relaying.

## Initial setup

Wait for the Headscale pod and its ingress certificate to become ready, then
create a Headscale user:

```sh
kubectl -n headscale exec deploy/headscale -- headscale users create hbarn
```

Create a one-time pre-authentication key when enrolling each device:

```sh
kubectl -n headscale exec deploy/headscale -- headscale users list
kubectl -n headscale exec deploy/headscale -- \
  headscale preauthkeys create --user <user-id>
```

On the NixOS server, enable the Tailscale NixOS service, set
`services.tailscale.useRoutingFeatures = "server";`, and enable IPv4 forwarding
with `boot.kernel.sysctl."net.ipv4.ip_forward" = "1";`. Enroll it as the LAN
subnet router:

```sh
sudo tailscale up \
  --login-server https://vpn.barnabash.com \
  --authkey <one-time-key> \
  --advertise-routes=192.168.0.0/24 \
  --accept-dns=false
```

Approve the route in Headscale:

```sh
kubectl -n headscale exec deploy/headscale -- \
  headscale nodes list-routes
kubectl -n headscale exec deploy/headscale -- \
  headscale nodes approve-routes --identifier <node-id> \
  --routes 192.168.0.0/24
```

Enroll Tailscale clients version 1.80 or newer against the same
`--login-server` URL. Headscale sends Pi-hole (`192.168.0.247`) as their DNS
server, so the existing `*.home.barnabash.com` records resolve over the
advertised LAN route. Only enroll devices you trust: the current tailnet policy
allows enrolled devices to communicate with one another.
