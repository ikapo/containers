# containers

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
