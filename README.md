# homelab

A small home server stack for media and related services, managed with Docker Compose.

This repository contains compose files and configuration for media services (Plex, Tautulli, Overseerr), download/automation tools (Sonarr, Radarr, qBittorrent, NZBGet, Prowlarr), a VPN gateway container (gluetun), and a Cloudflare Tunnel (`cloudflared`) used as a proxy for exposing selected services.

## Quick overview

- Location: `/workspaces/homelab`
- Key folders:
	- `media/` — plex, overseerr, cloudflared and media-related compose files
	- `apps/` — miscellaneous app compose files
	- `monitoring/` — monitoring stack compose files
	- `proxy/` — global reverse-proxy compose (cloudflared/tunnel)

Services of note (examples in this repo):

- Plex (container `plex`) — media server, configured in `media/plex/compose.yaml`
- Overseerr (container `overseerr`) — request management for Sonarr/Radarr
- Sonarr & Radarr — automation for TV & Movies (in `media/compose.yaml` and attached to `servarrnetwork`)
- cloudflared — Cloudflare Tunnel service used to expose services securely (configured in `media/plex/cloudflared/config.yml` mounted into the container)

## Network layout

This setup uses two Docker networks:

- `proxy` — a bridge network used by `cloudflared` and front-facing services (Plex, Overseerr, Tautulli). Cloudflared listens on this network and forwards traffic to containers by name.
- `servarrnetwork` — a custom network with static IPs where Sonarr / Radarr / Bazarr / Sonarr-like services live. The `servarrnetwork` is declared as `external: true` so multiple compose stacks can share it.


## How Overseerr reaches Sonarr & Radarr

Overseerr must be on the same Docker network as Sonarr/Radarr (or be able to reach them via routable IP). Recommended options:

1. Attach `overseerr` to `servarrnetwork` (declare `servarrnetwork: external: true` in the compose that runs Overseerr). Then use internal hostnames in Overseerr settings:
	 - Sonarr: `http://sonarr:8989`
	 - Radarr: `http://radarr:7878`

2. Alternatively use the static IPs assigned in `media/compose.yaml` (e.g. `172.39.0.3` for Sonarr, `172.39.0.4` for Radarr) but hostnames are more maintainable.

If Overseerr cannot reach Sonarr/Radarr after joining `servarrnetwork`:

- Confirm the network is declared `external: true` in both compose files.
- Confirm no firewall on the Docker host blocks communication between containers.
- Use `docker compose ps` and `docker network inspect servarrnetwork` to verify container attachments.

## Exposing Plex via Cloudflared Tunnel (recommended)

To expose Plex without opening ports on your router, use Cloudflare Tunnel (already configured here as the `cloudflared` container).

Example `ingress` snippet for `cloudflared/config.yml` (mounted into the container):

```yaml
tunnel: <your-tunnel-id>
credentials-file: /etc/cloudflared/<credentials>.json

ingress:
	- hostname: plex.huelin.dev
		service: http://plex:32400
	- service: http_status:404
```

Make sure:

- The `plex` container is attached to the same `proxy` network as `cloudflared` so `http://plex:32400` resolves.
- If you rely on UPnP/NAT-PMP auto-configuration of remote access, note that switching from `network_mode: host` to bridge mode will disable in-container UPnP. You can either map required UDP ports or keep host mode if you need full LAN discovery.

To bring Plex up and refresh cloudflared:

```bash
cd media
docker compose -f plex/compose.yaml up -d plex
docker compose -f plex/compose.yaml restart cloudflared
```

## Notes & troubleshooting

- Outbound network access: containers in bridge mode retain outbound internet access (Docker NAT). So Plex can still reach update servers or streaming providers.
- UPnP/SSDP/LAN discovery: map SSDP and related UDP ports (1900, 32410-32414) if you want local device discovery while using bridge mode.
- Overseerr -> Sonarr/Radarr auth: if you set API keys in Sonarr/Radarr, ensure Overseerr has the same key configured.
- If using gluetun (VPN) for downloaders, those services are often run using `network_mode: service:gluetun` or attached to the VPN to force traffic through the tunnel — be mindful that services on the VPN network may not be reachable from other bridge networks.

## File locations (quick reference)

- `media/compose.yaml` — main media compose containing Sonarr/Radarr/qBittorrent/gluetun and `servarrnetwork` definition.
- `media/plex/compose.yaml` — plex, overseerr, tautulli, cloudflared and `proxy` network.
- `media/plex/cloudflared/config.yml` — Cloudflare Tunnel ingress rules (mount location used by `cloudflared` container).

## Next steps / improvements

- Optionally add a small README for each subfolder with per-service notes (ports, volumes, env vars).
- Add a simple script to bootstrap networks and secrets (.env) or a Makefile to simplify common operations.
