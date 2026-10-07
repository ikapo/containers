# containers

## Choosing stacks per host

The same repo is checked out on every machine, and each machine runs a different
set of stacks. The stacks are picked per host in `.env` (gitignored), never by
editing tracked files, so `git status` stays clean and `git pull` never conflicts.

| File | What's in it |
| --- | --- |
| `docker-compose.yml` | Shared base: the pinned `containers_default` network and watchtower. Always listed first. |
| `compose.network.yml` | pihole, cloudflared, wireguard, nginx, dash |
| `compose.media.yml` | jellyfin, sonarr/radarr, prowlarr, transmission, autobrr, profilarr, seerr |
| `compose.apps.yml` | homepage, vaultwarden |
| `compose.hermes.yml` | hermes + obsidian-sync. Entry point that loads `.env.hermes`; the services are in `compose.hermes.services.yml`. |

Set `COMPOSE_FILE` in the host's `.env` to the files it should run, separated by `:`,
with `docker-compose.yml` first. For example, a host that runs only Hermes:

```sh
COMPOSE_FILE=docker-compose.yml:compose.hermes.yml
```

and a host that runs everything except Hermes:

```sh
COMPOSE_FILE=docker-compose.yml:compose.network.yml:compose.media.yml:compose.apps.yml
```

Compose reads `COMPOSE_FILE` from `.env` automatically, so plain `docker compose up -d`
(or `dc up -d`) starts only that host's stacks. Check which services a host will run with:

```sh
docker compose config --services
```

### Moving a stack to another host

1. Copy its runtime data, which isn't in git. For Hermes that's `configs/hermes/`,
   `obsidian/` (or wherever `VAULT_HOST_PATH` points) and `.env.hermes`:
   ```sh
   rsync -aP configs/hermes <host>:~/containers/configs/
   rsync -aP .env.hermes <host>:~/containers/
   ```
2. On the new host, add the stack's file to `COMPOSE_FILE` and run `docker compose up -d`.
3. On the old host, stop it first, e.g. `docker compose -f compose.hermes.yml down`,
   then remove it from `COMPOSE_FILE`. A plain `docker compose up -d` doesn't stop
   containers of stacks that were removed from the list; `--remove-orphans` does.

### Gotchas

- **Don't comment out stacks in `docker-compose.yml`.** That's a per-host edit to a tracked
  file, which is exactly what `COMPOSE_FILE` replaces.
- **Hermes needs `compose.hermes.yml`, not `compose.hermes.services.yml`.** Listing the
  services file directly skips `.env.hermes`, and every Hermes variable comes out blank.
  `COMPOSE_ENV_FILES` set in `.env` doesn't work either; Compose ignores it there.
- **New hosts:** copy `.env.example` to `.env` and trim `COMPOSE_FILE`. Without it, Compose
  falls back to `docker-compose.yml` alone and only the network and watchtower exist.

## Host setup

### Free the 172.18.0.0/16 subnet

`containers_default` is pinned to `172.18.0.0/16` (`docker-compose.yml`), and pihole
and cloudflared use static IPs in it (`compose.network.yml`). If another network
already uses that range, `dc up -d` fails with
`invalid pool request: Pool overlaps with other one on this address space`.
Find the conflicting network:

```sh
docker network inspect $(docker network ls -q) --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}}{{end}}'
ip route | grep 172.18
```

If it's a leftover Docker network, remove it (or `docker network prune` to remove all
unused networks):

```sh
docker network rm <name>
```

### Free port 53 for pihole

Pihole binds host port 53, which `systemd-resolved`'s DNS stub holds by default
(`failed to bind host port 0.0.0.0:53/tcp: address already in use`).
Check what is listening:

```sh
sudo ss -tulpn | grep ':53 '
```

If it's `systemd-resolved`, disable it and point the host at 1.1.1.1:

```sh
sudo systemctl disable --now systemd-resolved
sudo rm /etc/resolv.conf          # symlink to the resolved stub
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

If NetworkManager is running, stop it from overwriting `/etc/resolv.conf`:

```sh
printf '[main]\ndns=none\n' | sudo tee /etc/NetworkManager/conf.d/dns-none.conf
sudo systemctl restart NetworkManager
```
