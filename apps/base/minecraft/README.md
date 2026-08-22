# Minecraft (Java) server

Uses the popular `itzg/minecraft-server` chart, running the PaperMC server type
for good performance + plugin support.

## Before it works

1. `eula: "TRUE"` is already set (accepts the Mojang EULA).
2. Set the RCON password: `sops apps/base/minecraft/secret.sops.yaml`.
3. Add your + friends' usernames to `ops`/`whitelist` in `helmrelease.yaml`
   (set `whitelist` non-empty and add `WHITELIST` to enforce it).

## Connecting

The server is exposed as a `LoadBalancer` on port `25565`. Find its IP:

```sh
kubectl -n minecraft get svc
```

Give friends that IP (and forward 25565/tcp on your router if they're remote).

## Bedrock clients

Add the GeyserMC plugin (via the chart's `pluginUrls`/`modrinthProjects`) to let
Bedrock/console players join, then expose UDP 19132.

## Tuning

The Pi should stay around `memory: 2G`. On pc-k8s the overlay can bump memory
and CPU for more players / render distance.
