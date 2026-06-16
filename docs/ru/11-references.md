# 11 — References

Upstream-документация и первичные источники для каждого компонента, используемого в этом флоте.

## Linux kernel

- [Documentation — VRF](https://docs.kernel.org/networking/vrf.html) — `l3mdev`, двойной прогон prerouting, sysctl-настройки.
- [Documentation — IP sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.rst) — `tcp_l3mdev_accept`, `rp_filter`, `src_valid_mark`.
- [LWN — L3 master device intro (2015)](https://lwn.net/Articles/658471/).
- [SKB drop reason enum](https://dxuuu.xyz/dropreason.html) — что означает `SKB_DROP_REASON_*`.

## nftables

- [Project wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page).
- [Atomic rule replacement](https://wiki.nftables.org/wiki-nftables/index.php/Atomic_rule_replacement).
- [Ruleset debug / tracing](https://wiki.nftables.org/wiki-nftables/index.php/Ruleset_debug/tracing).
- `nft(8)` man — `man nft`.

## firewalld

- [Project page](https://firewalld.org/).
- [Documentation](https://firewalld.org/documentation/).
- Используй `firewall-cmd --get-zone-of-interface` и `firewall-cmd --list-all-zones` для интроспекции.

## WireGuard

- [Whitepaper](https://www.wireguard.com/papers/wireguard.pdf).
- [Quickstart](https://www.wireguard.com/quickstart/).
- [`wg-quick(8)`](https://man7.org/linux/man-pages/man8/wg-quick.8.html) — формат конфиг-файла и семантика `Table=off`.

## FRR (Free Range Routing)

- [Documentation root](https://docs.frrouting.org/).
- [BGP guide](https://docs.frrouting.org/en/latest/bgp.html) — communities, route-maps, `import vrf`.
- [Zebra guide](https://docs.frrouting.org/en/latest/zebra.html) — `ip protocol bgp route-map`, FIB-фильтры.
- [VRF guide](https://docs.frrouting.org/en/latest/vrf.html) — семантика именно в FRR.
- [`frr-reload.py` source](https://github.com/FRRouting/frr/blob/master/tools/frr-reload.py).
- Известные issue:
  - [#15909 — kernel 6.6/6.7 cross-VRF leak](https://github.com/FRRouting/frr/issues/15909)
  - [#3850 — route leak lost after reboot](https://github.com/FRRouting/frr/issues/3850)
  - [#13561 — kernel routes not updated in zebra RIB](https://github.com/FRRouting/frr/issues/13561)
  - [#15706 — frr-reload watchdog at large prefix counts](https://github.com/FRRouting/frr/issues/15706)

## ocserv (OpenConnect VPN server)

- [Project page](https://ocserv.openconnect-vpn.net/).
- [`ocserv(8)` manual](https://ocserv.openconnect-vpn.net/ocserv.8.html).
- [Sample config](https://gitlab.com/openconnect/ocserv/-/raw/master/doc/sample.config) — источник истины по каждой директиве.
- [Source `tun.c`](https://gitlab.com/openconnect/ocserv/-/raw/master/src/tun.c) — объясняет, почему `ipv6-subnet-prefix=128` не назначает global v6 на tun.

## Podman / netavark

- [Podman documentation](https://docs.podman.io/).
- [netavark — nftables backend source](https://github.com/containers/netavark/blob/main/src/firewall/nft.rs) — объясняет gate `fib daddr type local`, ради обхода которого существует `vpn_dnat.nft`.

## Pi-hole + dnscrypt-proxy

- [Pi-hole documentation](https://docs.pi-hole.net/).
- [dnscrypt-proxy wiki](https://github.com/DNSCrypt/dnscrypt-proxy/wiki).

## RIPE Stat (georoute feed)

- [Data API](https://stat.ripe.net/docs/data_api).
- [Country resource list endpoint](https://stat.ripe.net/data/country-resource-list/data.json?resource=RU&v4_format=prefix).
- Бесплатно; терпим к каденсу 12 ч.

## Ansible

- [User guide](https://docs.ansible.com/ansible/latest/user_guide/index.html).
- [`ansible-vault`](https://docs.ansible.com/ansible/latest/vault_guide/index.html).
- [Module index](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html).
- Используемые коллекции:
  - [`ansible.posix.firewalld`](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html).
  - [`community.general`](https://docs.ansible.com/ansible/latest/collections/community/general/index.html).

## Go

- [Go documentation home](https://go.dev/doc/).
- [`pkg.go.dev/os` — `os.CreateTemp`](https://pkg.go.dev/os#CreateTemp).
- [`pkg.go.dev/net/netip`](https://pkg.go.dev/net/netip) — `Prefix`, `Addr`.
- [`pkg.go.dev/golang.org/x/sys/unix`](https://pkg.go.dev/golang.org/x/sys/unix) — `Flock`.
- [`golangci-lint`](https://golangci-lint.run/) — агрегатор линтеров.

## Networking RFCs

- [RFC 4193 — Unique Local IPv6 Unicast Addresses](https://www.rfc-editor.org/rfc/rfc4193) — используемая здесь ULA `fdf3:bb42:9fc6::/48`.
- [RFC 5737 — IPv4 Address Blocks Reserved for Documentation](https://www.rfc-editor.org/rfc/rfc5737).
- [RFC 6598 — IANA-Reserved IPv4 Prefix for Shared Address Space (CGNAT)](https://www.rfc-editor.org/rfc/rfc6598) — используемый для клиентских пулов `100.64.0.0/10`.
- [RFC 4364 — BGP/MPLS IP Virtual Private Networks](https://www.rfc-editor.org/rfc/rfc4364) — словарь VRF-lite.
- [RFC 7854 — BMP](https://www.rfc-editor.org/rfc/rfc7854) — read-only мониторинг BGP (упомянуто в research по control-plane либам).
- [RFC 4271 — BGP-4](https://www.rfc-editor.org/rfc/rfc4271).
- [RFC 6996 — Autonomous System Reservation for Private Use](https://www.rfc-editor.org/rfc/rfc6996) — используемый здесь диапазон 4-байтных private ASN.

## Референс-репозитории, которые стоит прочитать

- [google/nftables](https://github.com/google/nftables) — Go-библиотека для нативных nftables-транзакций (Phase 4 swap target).
- [vishvananda/netlink](https://github.com/vishvananda/netlink) — netlink-либа выбора в Go.
- [osrg/gobgp](https://github.com/osrg/gobgp) — embeddable BGP speaker (альтернатива FRR для use case'а prefix-injection).
- [tailscale/tailscale — `util/linuxfw/nftables_runner.go`](https://github.com/tailscale/tailscale) — production-пример атомарных nft-обновлений.
- [cilium/cilium — `pkg/datapath/linux/route/route_linux.go`](https://github.com/cilium/cilium/blob/main/pkg/datapath/linux/route/route_linux.go) — production PBR rule/route management.

## Project-internal links

- [`dantte-lp/georoute`](https://github.com/dantte-lp/georoute) — Go-бинарь, документированный в [09-georoute.md](09-georoute.md).
- Этот репозиторий — [`dantte-lp/polyexit`](https://github.com/dantte-lp/polyexit).
