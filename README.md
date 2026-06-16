# polyexit

Ansible-managed fleet of OpenConnect (`ocserv`) VPN exit nodes with per-country geographical routing.

- **Multi-country exits** — one host per country (RU, UZ, …) plus a world-default host (e.g. FI).
- **Geo-routing** — `georoute` Go binary feeds country prefix lists from RIPE Stat into `nftables` PBR sets and FRR BGP announcements.
- **VRF isolation** — VPN-client traffic lives in a Linux VRF (`vrf-vpn`, table 10), never leaking into the host's own FIB.
- **Symmetric mesh** — WireGuard tunnels between sibling exits, eBGP with 4-byte ASNs, sibling-pool return routes.
- **Zero per-host shell** — every host configured by `ansible-playbook playbooks/site.yml`.

## Quick start

```bash
git clone https://github.com/dantte-lp/polyexit.git
cd polyexit
dnf install -y ansible-core
ansible-galaxy collection install ansible.posix community.general

# point hosts.yml at your nodes, fill in inventory/host_vars/dev-NN.yml
vim inventory/hosts.yml
vim inventory/host_vars/dev-03.yml

# (optional) encrypt secrets
cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml
vim inventory/group_vars/vault.yml
ansible-vault encrypt inventory/group_vars/vault.yml

# dry-run
ansible-playbook playbooks/site.yml --check --diff

# apply
ansible-playbook playbooks/site.yml
```

## Documentation

[`docs/`](docs/) — full guide in English (canonical) and Russian (translation).

Start with [`docs/en/01-overview.md`](docs/en/01-overview.md) or [`docs/ru/01-overview.md`](docs/ru/01-overview.md).

## License

MIT — see [LICENSE](LICENSE).
