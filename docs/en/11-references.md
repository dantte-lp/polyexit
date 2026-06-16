# 11 — References

Upstream documentation and primary sources for every component used in this fleet.

## Linux kernel

- [Documentation — VRF](https://docs.kernel.org/networking/vrf.html) — `l3mdev`, double prerouting traversal, sysctl knobs.
- [Documentation — IP sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.rst) — `tcp_l3mdev_accept`, `rp_filter`, `src_valid_mark`.
- [LWN — L3 master device intro (2015)](https://lwn.net/Articles/658471/).
- [SKB drop reason enum](https://dxuuu.xyz/dropreason.html) — what `SKB_DROP_REASON_*` means.

## nftables

- [Project wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page).
- [Atomic rule replacement](https://wiki.nftables.org/wiki-nftables/index.php/Atomic_rule_replacement).
- [Ruleset debug / tracing](https://wiki.nftables.org/wiki-nftables/index.php/Ruleset_debug/tracing).
- `nft(8)` man — `man nft`.

## firewalld

- [Project page](https://firewalld.org/).
- [Documentation](https://firewalld.org/documentation/).
- Use `firewall-cmd --get-zone-of-interface` and `firewall-cmd --list-all-zones` to introspect.

## WireGuard

- [Whitepaper](https://www.wireguard.com/papers/wireguard.pdf).
- [Quickstart](https://www.wireguard.com/quickstart/).
- [`wg-quick(8)`](https://man7.org/linux/man-pages/man8/wg-quick.8.html) — config file format and `Table=off` semantics.

## FRR (Free Range Routing)

- [Documentation root](https://docs.frrouting.org/).
- [BGP guide](https://docs.frrouting.org/en/latest/bgp.html) — communities, route-maps, `import vrf`.
- [Zebra guide](https://docs.frrouting.org/en/latest/zebra.html) — `ip protocol bgp route-map`, FIB filters.
- [VRF guide](https://docs.frrouting.org/en/latest/vrf.html) — semantics in FRR specifically.
- [`frr-reload.py` source](https://github.com/FRRouting/frr/blob/master/tools/frr-reload.py).
- Known issues:
  - [#15909 — kernel 6.6/6.7 cross-VRF leak](https://github.com/FRRouting/frr/issues/15909)
  - [#3850 — route leak lost after reboot](https://github.com/FRRouting/frr/issues/3850)
  - [#13561 — kernel routes not updated in zebra RIB](https://github.com/FRRouting/frr/issues/13561)
  - [#15706 — frr-reload watchdog at large prefix counts](https://github.com/FRRouting/frr/issues/15706)

## ocserv (OpenConnect VPN server)

- [Project page](https://ocserv.openconnect-vpn.net/).
- [`ocserv(8)` manual](https://ocserv.openconnect-vpn.net/ocserv.8.html).
- [Sample config](https://gitlab.com/openconnect/ocserv/-/raw/master/doc/sample.config) — the source of truth for every directive.
- [Source `tun.c`](https://gitlab.com/openconnect/ocserv/-/raw/master/src/tun.c) — explains why `ipv6-subnet-prefix=128` does not assign a global v6 to the tun.

## Podman / netavark

- [Podman documentation](https://docs.podman.io/).
- [netavark — nftables backend source](https://github.com/containers/netavark/blob/main/src/firewall/nft.rs) — explains the `fib daddr type local` gate that `vpn_dnat.nft` exists to work around.

## Pi-hole + dnscrypt-proxy

- [Pi-hole documentation](https://docs.pi-hole.net/).
- [dnscrypt-proxy wiki](https://github.com/DNSCrypt/dnscrypt-proxy/wiki).

## RIPE Stat (georoute feed)

- [Data API](https://stat.ripe.net/docs/data_api).
- [Country resource list endpoint](https://stat.ripe.net/data/country-resource-list/data.json?resource=RU&v4_format=prefix).
- Free; rate-tolerant at 12 h cadence.

## Ansible

- [User guide](https://docs.ansible.com/ansible/latest/user_guide/index.html).
- [`ansible-vault`](https://docs.ansible.com/ansible/latest/vault_guide/index.html).
- [Module index](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html).
- Collections used:
  - [`ansible.posix.firewalld`](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html).
  - [`community.general`](https://docs.ansible.com/ansible/latest/collections/community/general/index.html).

## Go

- [Go documentation home](https://go.dev/doc/).
- [`pkg.go.dev/os` — `os.CreateTemp`](https://pkg.go.dev/os#CreateTemp).
- [`pkg.go.dev/net/netip`](https://pkg.go.dev/net/netip) — `Prefix`, `Addr`.
- [`pkg.go.dev/golang.org/x/sys/unix`](https://pkg.go.dev/golang.org/x/sys/unix) — `Flock`.
- [`golangci-lint`](https://golangci-lint.run/) — linter aggregator.

## Networking RFCs

- [RFC 4193 — Unique Local IPv6 Unicast Addresses](https://www.rfc-editor.org/rfc/rfc4193) — the `fdf3:bb42:9fc6::/48` ULA used here.
- [RFC 5737 — IPv4 Address Blocks Reserved for Documentation](https://www.rfc-editor.org/rfc/rfc5737).
- [RFC 6598 — IANA-Reserved IPv4 Prefix for Shared Address Space (CGNAT)](https://www.rfc-editor.org/rfc/rfc6598) — the `100.64.0.0/10` used for client pools.
- [RFC 4364 — BGP/MPLS IP Virtual Private Networks](https://www.rfc-editor.org/rfc/rfc4364) — VRF-lite vocabulary.
- [RFC 7854 — BMP](https://www.rfc-editor.org/rfc/rfc7854) — read-only monitoring of BGP (mentioned in the research for control-plane libs).
- [RFC 4271 — BGP-4](https://www.rfc-editor.org/rfc/rfc4271).
- [RFC 6996 — Autonomous System Reservation for Private Use](https://www.rfc-editor.org/rfc/rfc6996) — the 4-byte private ASN range used here.

## Reference repositories worth reading

- [google/nftables](https://github.com/google/nftables) — Go library for native nftables transactions (Phase 4 swap target).
- [vishvananda/netlink](https://github.com/vishvananda/netlink) — the netlink lib of choice in Go.
- [osrg/gobgp](https://github.com/osrg/gobgp) — embeddable BGP speaker (alternative to FRR for the prefix-injection use case).
- [tailscale/tailscale — `util/linuxfw/nftables_runner.go`](https://github.com/tailscale/tailscale) — production example of atomic nft updates.
- [cilium/cilium — `pkg/datapath/linux/route/route_linux.go`](https://github.com/cilium/cilium/blob/main/pkg/datapath/linux/route/route_linux.go) — production PBR rule/route management.

## Project-internal links

- [`dantte-lp/georoute`](https://github.com/dantte-lp/georoute) — the Go binary documented in [09-georoute.md](09-georoute.md).
- This repository — [`dantte-lp/polyexit`](https://github.com/dantte-lp/polyexit).
