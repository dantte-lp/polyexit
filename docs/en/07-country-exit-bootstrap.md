# 07 — Adding a country exit

End-to-end bring-up of a new country exit node, illustrated with Uzbekistan (`dev-05`, ISO `UZ`, ISO-numeric `860`).

## Prerequisites

| Item                 | Required value                                                                 |
|----------------------|--------------------------------------------------------------------------------|
| VPS in the country   | static public IPv4; IPv6 nice to have                                          |
| Root SSH access      | key-only; the controller's pubkey added to `/root/.ssh/authorized_keys`        |
| Provider not blocking| TCP/443 (mandatory) and UDP/443 (preferred), plus the chosen WG port           |
| 30 GB disk           | root partition                                                                 |
| 1 GB RAM             | minimum; 2 GB comfortable                                                      |

## Pre-allocate identifiers

```text
site:           uz
site_octet:     5
ASN:            4200000005             # = 4200000000 + site_octet (4-byte private)
v4 pool:        100.64.5.0/24
v6 pool:        fdf3:bb42:9fc6:5::/64
BGP router-id:  10.255.0.5 / fdf3:bb42:9fc6:ff00::5
fwmark:         0x205
PBR table:      105
community:      64512:2860             # = 64512:2<ISO-numeric>
WG iface name:  wg103                  # on dev-03 = wg<sibling_site_octet * 100>
                wg0                    # on dev-05 — single tunnel toward dev-03
WG /31 slot:    169.254.255.6/7        # pair (3,5) — see addressing scheme
WG v6 link-local: fe80::3 (dev-03 side), fe80::5 (dev-05 side)
```

The `0x205` / table 105 / community `64512:2860` are derived from `site_octet` and `iso_numeric` — the formulas are in [Inventory](04-inventory.md).

## Step 1 — provision VPS

Anything that gives you root SSH on Oracle Linux 10 / Rocky 10 / Alma 10. Reference: [Oracle Linux 10 install guide](https://docs.oracle.com/en/operating-systems/oracle-linux/10/install/).

After first login:

```bash
dnf install -y python3            # required by Ansible for `raw` fallback to play
```

That is all that needs to happen out-of-band — the rest of the build is in the playbook.

## Step 2 — inventory

Edit [`inventory/hosts.yml`](../../inventory/hosts.yml):

```yaml
exits-country-local:
  hosts:
    dev-05:
      ansible_host: 213.230.X.X        # ← real UZ VPS IP
      ansible_user: root
      ansible_port: 22
      ansible_connection: ssh
```

Edit [`inventory/host_vars/dev-05.yml`](../../inventory/host_vars/dev-05.yml) — most fields are already filled with UZ placeholders. Update the four `TBD` lines:

```yaml
public_v4: 213.230.X.X
public_v6: ""                          # UZ commercial v6 is poor — leave empty
host_v4_uplink: ens1                   # check with `ip route show default`
host_v4_gw: 213.230.X.1                # the provider's GW
```

## Step 3 — controller can reach it

```bash
ansible-playbook playbooks/site.yml --limit dev-05 -m ping
# expected:
# dev-05 | SUCCESS => {"ansible_facts": {...}, "ping": "pong"}
```

If this fails, fix SSH before going further.

## Step 4 — WireGuard tunnel to dev-03 (sibling-of-record)

```bash
# on dev-05
dnf install -y wireguard-tools
mkdir -p /etc/wireguard
umask 077
wg genkey | tee /etc/wireguard/wg0.key | wg pubkey > /etc/wireguard/wg0.pub
wg genpsk > /etc/wireguard/wg0.psk      # shared with dev-03

# on dev-03
wg genkey | tee /etc/wireguard/wg103.key | wg pubkey > /etc/wireguard/wg103.pub
# use the SAME psk on both sides
```

Configure both sides — dev-05's `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 169.254.255.7/31
Address = fdf3:bb42:9fc6:ffff::5/127
PrivateKey = <dev-05 private>
ListenPort = 31518
Table = off
PostUp = ip -4 route add 169.254.255.6/32 dev wg0
PostUp = ip -6 route add fdf3:bb42:9fc6:ffff::4/128 dev wg0

[Peer]
PublicKey = <dev-03 pubkey>
PresharedKey = <psk>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = 82.26.171.163:31518
PersistentKeepalive = 25
```

dev-03's `/etc/wireguard/wg103.conf` is the mirror image. Bring both up:

```bash
systemctl enable --now wg-quick@wg0    # on dev-05
systemctl enable --now wg-quick@wg103  # on dev-03
ping -c2 169.254.255.6                 # from dev-05 — should reply
```

## Step 5 — FRR neighbor sections

Add to dev-03's `/etc/frr/frr.conf` in the `router bgp 64512` block:

```text
 neighbor 169.254.255.7 remote-as 4200000005
 neighbor 169.254.255.7 description dev-05-uz
 neighbor fe80::5%wg103 remote-as 4200000005
 neighbor fe80::5%wg103 description dev-05-uz-v6
 !
 address-family ipv4 unicast
  neighbor 169.254.255.7 activate
  neighbor 169.254.255.7 route-map FROM-DEV-05 in
  neighbor 169.254.255.7 route-map TO-DEV-05 out
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor fe80::5%wg103 activate
  neighbor fe80::5%wg103 route-map V6-FROM-DEV-05 in
  neighbor fe80::5%wg103 route-map V6-TO-DEV-05 out
 exit-address-family
```

Plus the matching route-maps with `match community CL-CC-UZ` (community `64512:2860`) and `set local-preference 200` on inbound, `set community 64512:2860 additive` on the announced country prefixes.

dev-05's `frr.conf` will be templated by the future FRR Ansible role; for now copy the structure from dev-04 and adjust ASN / neighbor IPs.

Test:

```bash
vtysh -c 'show bgp summary'           # peer should be Established
```

## Step 6 — apply the playbook

```bash
ansible-playbook playbooks/site.yml --limit dev-05
```

Expected effects on dev-05:

- `common`, `vrf-vpn`, `nft-vpn`, `ocserv`, `georoute` roles applied
- `/usr/local/bin/georoute` deployed
- `pbr.nft` loaded with empty `uz_v4` / `uz_v6` sets
- `georoute@uz.timer` enabled (will fire 5 min after boot or on next run)

## Step 7 — fire georoute once manually (don't wait 5 min)

```bash
ssh root@dev-05 systemctl start georoute@uz.service
ssh root@dev-05 journalctl -u georoute@uz.service -n 10 --no-pager
# expected last line:
#   georoute[UZ] ... nft sets updated (uz_v4=162 uz_v6=48)
#   georoute[UZ] ... frr-reload completed
```

## Step 8 — verify BGP propagation to dev-03

```bash
# on dev-03
vtysh -c 'show bgp summary' | grep dev-05      # peer Established, PfxRcd ≈ 162+48
vtysh -c 'show bgp ipv4 unicast community 64512:2860' | head
ip -4 route show table 10 | grep wg103 | head  # UZ prefixes via wg103
```

## Step 9 — end-to-end test from a real VPN client

Connect to `news.infra4.dev` or `docs.infra4.dev` (Cloudflare multi-A round-robin to dev-03 / dev-04 / dev-05).

```bash
# from the VPN client
curl -4 ifconfig.io                          # expect Finland IP (dev-03 default exit)
curl -4 -H "Host: ifconfig.io" 5.255.255.77  # this RU IP — expect Moscow exit IP
curl -4 -H "Host: ifconfig.io" 213.230.0.1   # UZ IP — expect Tashkent exit IP
```

`ifconfig.io` returns the source IP — you can directly read which exit the traffic took.

## Step 10 — alert on BGP drop

Out of scope of this playbook, but recommended:

```bash
# check every 5 min — page if the country prefix count drops by >50%
vtysh -c 'show bgp ipv4 unicast community 64512:2860' | grep -c '^[*>]'
```

Push the number to your monitoring system (Prometheus, Zabbix, Healthchecks — your call).

## What to read next

- [User management](08-user-management.md) — make the new exit usable.
- [Troubleshooting](10-troubleshooting.md) — when one of the steps above breaks.
