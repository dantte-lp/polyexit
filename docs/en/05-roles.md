# 05 — Roles

## Catalog

| Role         | Always applied | Conditional                                  | Touches                                                                         |
|--------------|----------------|----------------------------------------------|---------------------------------------------------------------------------------|
| `common`     | yes            | —                                            | base packages, `/etc/sysctl.d/91-router.conf`                                    |
| `vrf-vpn`    | yes            | —                                            | `vrf-vpn.service`, `infra-loopback.service`, firewalld zones, kernel `vrf` module |
| `nft-vpn`    | yes            | `vpn_dnat.nft` skipped if `!secure_dns_enabled` | `/etc/nft.d/mss_clamp.nft`, `/etc/nft.d/vpn_dnat.nft`, two systemd one-shots    |
| `secure-dns` | no             | `secure_dns_enabled: true`                   | podman compose at `/opt/project/repositories/secure-dns-vpn`, socat v6 service  |
| `ocserv`     | yes            | —                                            | `ocserv.conf`, `connect-vrf.sh`, `occtl reload` handler                          |
| `georoute`   | no             | `country is defined`                         | `/usr/local/bin/georoute`, `pbr.nft`, `georoute@<cc>.service|timer`              |

`site.yml` chains them in this exact order — `vrf-vpn` before `nft-vpn` because the DNAT chain references the vrf master device; `ocserv` last because `occtl reload` needs the connect-script in place.

## `common`

Source: [`roles/common/`](../../roles/common/)

What it does:

- Installs every package in `base_packages` (from `group_vars/all.yml`).
- Ensures `firewalld` is enabled and started.
- Writes `/etc/sysctl.d/91-router.conf` with kernel knobs the rest of the fleet depends on.

Key sysctl knobs the role enforces (see [`templates/91-router.conf.j2`](../../roles/common/templates/91-router.conf.j2)):

| Knob                                  | Value | Why                                                                  |
|---------------------------------------|-------|----------------------------------------------------------------------|
| `net.ipv4.conf.all.rp_filter`         | `2`   | loose mode — needed for VRF return paths                              |
| `net.ipv4.conf.all.src_valid_mark`    | `1`   | required for fwmark-based PBR                                         |
| `net.ipv4.tcp_l3mdev_accept`          | `1`   | cross-VRF socket lookup for both v4 and v6 (despite the `ipv4` prefix)|
| `net.ipv4.udp_l3mdev_accept`          | `1`   | same — both families                                                  |
| `net.ipv4.raw_l3mdev_accept`          | `1`   | cross-VRF ICMP                                                        |
| `net.ipv4.ip_forward`                 | `1`   | host is a router                                                      |
| `net.ipv6.conf.all.forwarding`        | `1`   | same for v6                                                           |

Reference: [Documentation/networking/ip-sysctl.rst](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.rst).

## `vrf-vpn`

Source: [`roles/vrf-vpn/`](../../roles/vrf-vpn/)

What it does:

1. Loads the `vrf` kernel module (one-shot on apply + module-load.d for boot).
2. Writes `infra-loopback.service` — assigns BGP router-id `/32` and (if `secure_dns_enabled`) DNS service-IPs to `lo`.
3. Writes `vrf-vpn.service` — creates the `vrf-vpn` master, the unreachable backstop in table 10, return routes in main, sibling-pool return routes in main, cross-VRF default in table 10, and (if `secure_dns_enabled`) the v6 ip-rule for DNS.
4. Adds `vrf-vpn` and `vpns+` to firewalld zone `trusted`.

`vrf-vpn.service` `ExecStart` chain (templated — see [`templates/vrf-vpn.service.j2`](../../roles/vrf-vpn/templates/vrf-vpn.service.j2)):

```ini
ExecStart=/usr/sbin/ip link add vrf-vpn type vrf table 10
ExecStart=/usr/sbin/ip link set vrf-vpn up
ExecStart=/usr/sbin/ip -4 route replace unreachable default table 10 metric 4278198272
ExecStart=/usr/sbin/ip -6 route replace unreachable default table 10 metric 4278198272
ExecStart=/usr/sbin/ip -4 route replace 100.64.X.0/24 dev vrf-vpn          # own client pool — main FIB return
ExecStart=/usr/sbin/ip -6 route replace fdf3:bb42:9fc6:X::/64 dev vrf-vpn  # same v6
ExecStart=/usr/sbin/ip -4 route replace 100.64.Y.0/24 via 169.254.255.Z dev wgN  # sibling pool return
ExecStart=/usr/sbin/ip -6 route replace fdf3:bb42:9fc6:Y::/64 via fe80::N dev wgN
ExecStart=/usr/sbin/ip -4 route replace 172.20.0.0/24 dev podman4 table 10        # only if secure_dns
ExecStart=/usr/sbin/ip -6 rule add iif "vpns+" to fdf3:…:Y::53/128 lookup main pref 900   # only if secure_dns
ExecStart=/usr/sbin/ip -4 route replace default via <gw> dev <iface> table 10
ExecStart=/usr/sbin/ip -6 route replace default via <gw> dev <iface> table 10
```

Reference: [Linux VRF docs](https://docs.kernel.org/networking/vrf.html).

## `nft-vpn`

Source: [`roles/nft-vpn/`](../../roles/nft-vpn/)

What it does:

- Always: deploys `/etc/nft.d/mss_clamp.nft` + a `nft-mss-clamp.service` one-shot that loads it at boot.
- If `secure_dns_enabled`: deploys `/etc/nft.d/vpn_dnat.nft` + matching service.

`mss_clamp.nft` clamps TCP MSS on the inter-site WG interface (templated `{{ wg_sibling_iface }}`) — needed because RU sites in particular drop ICMP-PTB and would otherwise blackhole large packets.

```nft
table inet mss_clamp {
    chain forward {
        type filter hook forward priority mangle; policy accept;
        oifname "wg102" tcp flags syn tcp option maxseg size set 1300 counter
        oifname "wg102" meta l4proto tcp ip6 nexthdr tcp tcp flags syn tcp option maxseg size set 1240 counter
        iifname "wg102" tcp flags syn tcp option maxseg size set 1300 counter
        iifname "wg102" meta l4proto tcp ip6 nexthdr tcp tcp flags syn tcp option maxseg size set 1240 counter
    }
}
```

`vpn_dnat.nft` is the workaround for netavark's cross-VRF DNAT gap:

```nft
table inet vpn_dnat {
    chain prerouting {
        type nat hook prerouting priority -101; policy accept;
        iifname "vpns*" ip daddr 100.64.X.53 udp dport 53 counter dnat ip to 172.20.0.20:53
        iifname "vpns*" ip daddr 100.64.X.53 tcp dport 53 counter dnat ip to 172.20.0.20:53
        iifname "vrf-vpn" ip daddr 100.64.X.53 udp dport 53 counter dnat ip to 172.20.0.20:53
        iifname "vrf-vpn" ip daddr 100.64.X.53 tcp dport 53 counter dnat ip to 172.20.0.20:53
    }
}
```

Why priority `-101` (one before netavark's `-100`): netavark gates DNAT on `fib daddr type local` which is false for the service-IP from `vrf-vpn`'s perspective. Hooking earlier with explicit iif lets the packet reach Pi-hole.

Reference: [nftables wiki — atomic rule replacement](https://wiki.nftables.org/wiki-nftables/index.php/Atomic_rule_replacement); [netavark nft DNAT source](https://github.com/containers/netavark/blob/main/src/firewall/nft.rs).

## `secure-dns`

Source: [`roles/secure-dns/`](../../roles/secure-dns/)

Skipped when `secure_dns_enabled` is false (default for country-exit hosts).

What it does:

- Installs `podman` + `podman-compose`.
- Writes `compose.yaml` to `/opt/project/repositories/secure-dns-vpn/` with Pi-hole bound on host-IP `<v4 service-IP>` and a private bridge `172.20.0.0/24`.
- Writes `.env` with secrets from vault.
- Writes two systemd units — `dns-v6-proxy.service` and `dns-v6-proxy-tcp.service` — that run `socat` on the v6 service-IP forwarding to `127.0.0.1:53` (because netavark does not implement IPv6 DNAT).
- Adds `podman4` bridge to firewalld zone `trusted`.

DNS path summary:

```text
client → vpns0 → nft DNAT (vpn_dnat) → podman4 bridge → Pi-hole 172.20.0.20:53
                                                              ↓
                                                       dnscrypt-proxy 172.20.0.10:5053
                                                              ↓
                                                       Cloudflare / Quad9 via DoH
```

References: [Pi-hole](https://docs.pi-hole.net/), [dnscrypt-proxy](https://dnscrypt.info/).

## `ocserv`

Source: [`roles/ocserv/`](../../roles/ocserv/)

What it does:

- Installs `ocserv` (`dnf install ocserv`).
- Writes `/etc/ocserv/connect-vrf.sh` — the connect-script hook that enslaves the per-client `vpns*` tunnel into `vrf-vpn` and installs the client's v6 `/128` in **table 10**.
- Renders `/etc/ocserv/ocserv.conf` from `templates/ocserv.conf.j2` using `ocserv_vhosts` + `push_dns_resolvers` + per-host pools.
- Validates the config with `ocserv -t` before writing.
- Notifies handler `occtl reload` (graceful — does not drop active sessions).

The `connect-vrf.sh` body is short but load-bearing:

```bash
#!/bin/bash
DEV="${IFNAME:-$DEVICE}"
[ -n "$DEV" ] || exit 0
/usr/sbin/ip link set "$DEV" master vrf-vpn 2>/dev/null || true
if [ -n "${IPV6_REMOTE:-}" ]; then
    /usr/sbin/ip -6 route replace "${IPV6_REMOTE}/128" dev "$DEV" table 10 2>/dev/null || true
fi
logger -t ocserv-vrf "connected user=${USERNAME:-?} real=${IP_REAL:-?} iface=$DEV v4=${IP_REMOTE:-?} v6=${IPV6_REMOTE:-?} → vrf-vpn"
exit 0
```

Why the `table 10` is explicit on the v6 add: ocserv adds the route via `SIOCADDRT`, which defaults to `RT_TABLE_MAIN` regardless of the slave's VRF master — and the v6 reply path needs it in vrf-vpn's table 10.

### Carrying compat directives

The template ships these legacy-compat directives on by default, kept because production clients still include older AnyConnect-derived devices:

| Directive                       | Default | When to disable                                                         |
|---------------------------------|---------|-------------------------------------------------------------------------|
| `cisco-client-compat = true`    | on      | All clients are OpenConnect ≥ 8.0; AnyConnect Mobile ≥ 4.x              |
| `cisco-svc-client-compat = true`| on      | Same — deprecated in newer ocserv builds; check `ocserv -t` warnings    |
| `dtls-legacy = true`            | on      | All clients negotiate DTLS 1.2 (`gnutls-cli --priority=NORMAL ...`)      |
| `compression = false`           | off     | Keep off — DTLS already compresses; CRIME-class risks                   |
| `try-mtu-discovery = true`      | on      | Set to `false` + pin `mtu` when PMTUD across the VRF boundary misbehaves |

Audit reachable clients with `occtl show users` — the `dtls-cipher` column tells you which sessions still need DTLS-legacy paths.

Reference: [`ocserv` manual](https://ocserv.openconnect-vpn.net/ocserv.8.html), [`ocserv` sample config](https://gitlab.com/openconnect/ocserv/-/raw/master/doc/sample.config).

## `georoute`

Source: [`roles/georoute/`](../../roles/georoute/)

Skipped unless `country` is defined in host vars.

What it does:

1. Builds the `georoute` Go binary on the controller (`delegate_to: localhost`) and copies it to `/usr/local/bin/georoute` on the target.
2. Writes `/etc/nft.d/pbr.nft` declaring the country set + the prerouting/output chains that set the country `fwmark`.
3. Writes `nft-pbr.service` to load `pbr.nft` on boot.
4. Adds the `ip rule fwmark <country.fwmark> lookup <country.pbr_table>` for v4 and v6.
5. Sets the per-country PBR table default to the host's local uplink.
6. Writes `/etc/georoute/<cc>.env` and the systemd template `georoute@.service` + `georoute@.timer`.
7. Enables `georoute@<cc>.timer`.

Timer cadence: `OnBootSec=5min`, `OnUnitActiveSec=12h`, `RandomizedDelaySec=30min`.

The `georoute` binary itself is documented separately in [09-georoute.md](09-georoute.md).

## What to read next

- [Deployment](06-deployment.md) — first apply, common flags.
- [Adding a country exit](07-country-exit-bootstrap.md) — production bring-up checklist.
