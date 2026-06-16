# 04 — Inventory

## Files

```text
inventory/
├── hosts.yml                    # group + host membership
├── group_vars/
│   ├── all.yml                  # shared across all hosts
│   └── vault.yml.example        # template for secrets (encrypt before commit)
└── host_vars/
    ├── dev-03.yml               # FI world-default
    ├── dev-04.yml               # RU country-exit
    └── dev-05.yml               # UZ country-exit (placeholder)
```

## `hosts.yml` — group hierarchy

```yaml
all:
  children:
    exits:
      children:
        exits-world:
          hosts:
            dev-03:
              ansible_host: 192.0.2.3
              ansible_connection: local
        exits-country-local:
          hosts:
            dev-04:
              ansible_host: 198.51.100.4
              ansible_connection: ssh
            dev-05:
              ansible_host: "{{ public_v4 }}"
              ansible_connection: ssh
```

- **`exits-world`** — hosts whose VPN clients leave through the host's own uplink.
- **`exits-country-local`** — hosts whose VPN clients leave through their own uplink *only* for their country's prefixes; all other traffic transits WireGuard to a world-default sibling.

A host can be in only one of these subgroups.

## `group_vars/all.yml`

Variables shared across the entire fleet. Examples:

```yaml
bgp_asn: 64512                          # legacy ASN for dev-03 (Mikrotik compatibility)
ula_prefix: "fdf3:bb42:9fc6::/48"       # RFC 4193 ULA — sliced /64-per-host
vrf_vpn_table: 10
vrf_vpn_name: vrf-vpn

ocserv:
  max_clients: 128
  dpd: 60
  mobile_dpd: 300
  keepalive: 600
  rekey_time: 172800
  no_routes:
    - "192.168.0.0/255.255.0.0"
    - "10.0.0.0/255.0.0.0"
    - "172.16.0.0/255.240.0.0"

base_packages:
  - nftables
  - firewalld
  # …
```

## `host_vars/dev-NN.yml` — per-host model

Every host has one of these. The shape is identical; only values differ. Required fields:

```yaml
# Identity + reachability
public_v4: 198.51.100.4          # host's own public v4
public_v6: "2001:db8:ru::2"     # host's own public v6 (or "" if v4-only)
site: ru                           # short slug
role: country-exit                 # world-default | country-exit
site_octet: 4                      # 1-byte deterministic key — many vars derive from it

# Host's own uplinks
host_v4_uplink: ens1
host_v4_gw: 198.51.100.1
host_v6_uplink: sit1

# VRF default exit — where VPN clients in vrf-vpn go for non-country traffic
v4_uplink_iface: wg0               # country-exit: wg to world-default; world-default: ens1
v4_uplink_gw: 169.254.255.0
v6_uplink_iface: wg0
v6_uplink_gw: "fe80::3"
wg_sibling_iface: wg0              # name of the inter-site WG iface on THIS host

# Client pools — deterministic from site_octet
v4_pool_cidr: 100.64.4.0/24
v4_pool_netmask: 255.255.255.0
v4_pool_network: 100.64.4.0
v4_pool_gw: 100.64.4.1
v6_pool_cidr: "fdf3:bb42:9fc6:4::/64"
v6_pool_gw: "fdf3:bb42:9fc6:4::1"

# Pi-hole service-IPs (only used if secure_dns_enabled=true)
dns_service_v4: 100.64.4.53
dns_service_v6: "fdf3:bb42:9fc6:4::53"
secure_dns_enabled: false
push_dns_resolvers:
  - "9.9.9.9"
  - "2606:4700:4700::1111"

# BGP
bgp_router_id_v4: 10.255.0.4
bgp_router_id_v6: "fdf3:bb42:9fc6:ff00::4"

# Sibling site for transit-return routes
sibling_v4_pool: 100.64.3.0/24
sibling_v6_pool: "fdf3:bb42:9fc6:3::/64"
wg_sibling_peer_v4: 169.254.255.0
wg_sibling_peer_v6: "fe80::3"

# ocserv vhosts
ocserv_vhosts:
  - name: docs.infra4.dev
    is_default: true
    user_profile: docs.xml
  - name: news.infra4.dev
    is_default: false
    user_profile: news.xml

# Country block — drives the georoute role
country:
  iso2: RU
  iso_numeric: 643
  bgp_community: "64512:201"          # legacy for RU; new countries use "64512:2<iso_numeric>"
  route_map: MARK-RU-EXIT
  nft_set_prefix: ru
  feed_url: "https://stat.ripe.net/data/country-resource-list/data.json?resource=RU&v4_format=prefix"
  fwmark: "0x201"
  pbr_table: 100
```

## Naming derivations

Many host-level vars are deterministic from `site_octet`:

| Derived variable        | Formula                                       | Example (`site_octet: 4`)             |
|-------------------------|-----------------------------------------------|---------------------------------------|
| `v4_pool_cidr`          | `100.64.<octet>.0/24`                         | `100.64.4.0/24`                       |
| `v6_pool_cidr`          | `fdf3:bb42:9fc6:<octet>::/64`                 | `fdf3:bb42:9fc6:4::/64`               |
| `bgp_router_id_v4`      | `10.255.0.<octet>`                            | `10.255.0.4`                          |
| `bgp_router_id_v6`      | `fdf3:bb42:9fc6:ff00::<octet>`                | `fdf3:bb42:9fc6:ff00::4`              |
| `country.fwmark`        | `0x200 | site_octet` (recommended for new)    | new: `0x205`; legacy RU stays `0x201` |
| `country.pbr_table`     | `100 + site_octet` (recommended for new)      | new: `105`; legacy RU stays `100`     |

> Legacy hosts (dev-04) keep their original `0x201` / table 100 for backward compatibility. New hosts should follow the formulas.

## Secrets — `vault.yml`

Encrypt with `ansible-vault`:

```bash
cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml
$EDITOR inventory/group_vars/vault.yml
ansible-vault encrypt inventory/group_vars/vault.yml
echo 'YOUR_VAULT_PASSWORD' > ~/.ansible_vault_pass
chmod 600 ~/.ansible_vault_pass
# uncomment the `vault_password_file` line in ansible.cfg
```

Recognized vars inside `vault.yml`:

```yaml
vault_cf_api_token: "..."             # Cloudflare DNS API token (for cert deploy-hook)
vault_pihole_web_password: "..."      # Pi-hole web admin password (secure-dns role)
vault_bgp_md5: "..."                  # eBGP MD5 password (FRR — not yet templated)
```

[`group_vars/all.yml`](../../inventory/group_vars/all.yml) references these as `vault_*` so it is obvious where they came from. Documentation reference: [ansible-vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

## Adding a new host

1. Allocate a `site_octet`. Convention: 3 = FI, 4 = RU, 5 = UZ, 6 = KZ, etc.
2. Create `inventory/host_vars/dev-NN.yml` following the model above.
3. Add to `hosts.yml` under the right group (`exits-world` or `exits-country-local`).
4. Run [the bootstrap checklist](07-country-exit-bootstrap.md).

## What to read next

- [Roles](05-roles.md) — what every role does and what it touches.
- [Deployment](06-deployment.md) — first apply.
