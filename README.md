# infra-config

Ansible-managed configuration for the OpenConnect (`ocserv`) exit fleet.

Two hosts today; the structure assumes more are coming. Each exit node holds:

- A Linux VRF (`vrf-vpn`, table 10) that isolates VPN client traffic from the
  host's own routing table.
- An ocserv worker pool per vhost (`docs.infra4.dev` default, `news.infra4.dev`
  geo-balanced via Cloudflare multi-A).
- A FRR eBGP session over WireGuard `wg102` to the sibling site, used for
  geo-aware exit selection.
- (Optionally) a `secure-dns-vpn` podman stack (Pi-hole + dnscrypt-proxy +
  IPv6 socat proxy) bound to a host-local service-IP `<v4_pool>.53` and
  `<v6_pool>::53`. When the stack is absent the playbook pushes public
  resolvers in `ocserv.conf` instead.

The cross-VRF DNS-reply path is the load-bearing fragility — see the comments
in `roles/vrf-vpn/templates/vrf-vpn.service.j2` and
`roles/nft-vpn/templates/vpn_dnat.nft.j2` for why we hook `prerouting` at
priority `-101` and add the `ip rule iif vpns+ ... lookup main pref 900`
escape.

## Quick start

```bash
# from dev-03 (this repo lives at /opt/project/repositories/infra-config/)
cd /opt/project/repositories/infra-config

# 1. create the vault password file once
umask 077; echo 'YOUR_VAULT_PASSWORD' > ~/.ansible_vault_pass

# 2. create the encrypted vault (copy example, fill in real secrets, encrypt)
cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml
vim inventory/group_vars/vault.yml
ansible-vault encrypt inventory/group_vars/vault.yml

# 3. install Ansible if needed
dnf install -y ansible-core

# 4. sanity check inventory + connectivity
ansible all -m ping

# 5. dry-run the full site
ansible-playbook playbooks/site.yml --check --diff

# 6. apply for real
ansible-playbook playbooks/site.yml

# Apply to one host only
ansible-playbook playbooks/site.yml --limit dev-04

# Apply one role only
ansible-playbook playbooks/site.yml --tags vrf-vpn
```

## Roles

| Role          | Purpose                                                          |
|---------------|------------------------------------------------------------------|
| `common`      | sysctls, base packages, firewalld, timezone                      |
| `vrf-vpn`     | VRF master + table 10 routes + ip rules + infra-loopback         |
| `nft-vpn`     | cross-VRF DNAT + MSS clamp on wg102                              |
| `secure-dns`  | Pi-hole + dnscrypt + socat v6 (only when `secure_dns_enabled`)   |
| `ocserv`      | ocserv.conf (per vhost), connect-vrf.sh, systemd ordering        |

## Host vars

The per-host variables in `inventory/host_vars/<host>.yml` are the only place
where per-site numbering lives (v4/v6 pools, BGP router-id, uplinks, DNS push,
feature flags). Roles and templates only consume those vars.

## Adding a new exit node

1. Append host stanza to `inventory/hosts.yml`.
2. Create `inventory/host_vars/<host>.yml` with `site_octet`, uplinks,
   `secure_dns_enabled`, etc.
3. Bootstrap SSH access (`ssh-copy-id`) and ensure Python on the target
   (EL10 has it by default).
4. `ansible-playbook playbooks/site.yml --limit <host>`.
