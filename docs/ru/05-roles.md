# 05 — Роли

## Каталог

| Роль         | Применяется всегда | Условно                                      | Трогает                                                                         |
|--------------|--------------------|----------------------------------------------|---------------------------------------------------------------------------------|
| `common`     | да                 | —                                            | базовые пакеты, `/etc/sysctl.d/91-router.conf`                                   |
| `vrf-vpn`    | да                 | —                                            | `vrf-vpn.service`, `infra-loopback.service`, зоны firewalld, модуль ядра `vrf`  |
| `nft-vpn`    | да                 | `vpn_dnat.nft` пропускается, если `!secure_dns_enabled` | `/etc/nft.d/mss_clamp.nft`, `/etc/nft.d/vpn_dnat.nft`, два systemd one-shot |
| `secure-dns` | нет                | `secure_dns_enabled: true`                   | podman compose в `/opt/project/repositories/secure-dns-vpn`, сервис socat v6    |
| `ocserv`     | да                 | —                                            | `ocserv.conf`, `connect-vrf.sh`, handler `occtl reload`                          |
| `georoute`   | нет                | `country is defined`                         | `/usr/local/bin/georoute`, `pbr.nft`, `georoute@<cc>.service|timer`              |

`site.yml` сцепляет их именно в этом порядке — `vrf-vpn` перед `nft-vpn`, потому что DNAT-цепочка ссылается на vrf master device; `ocserv` последним, потому что `occtl reload` требует connect-script на месте.

## `common`

Источник: [`roles/common/`](../../roles/common/)

Что делает:

- Ставит каждый пакет из `base_packages` (из `group_vars/all.yml`).
- Гарантирует, что `firewalld` enabled и стартован.
- Пишет `/etc/sysctl.d/91-router.conf` с настройками ядра, от которых зависит остальной флот.

Ключевые sysctl-настройки, которые роль фиксирует (смотри [`templates/91-router.conf.j2`](../../roles/common/templates/91-router.conf.j2)):

| Настройка                             | Значение | Зачем                                                                |
|---------------------------------------|----------|----------------------------------------------------------------------|
| `net.ipv4.conf.all.rp_filter`         | `2`      | loose mode — нужен для return-путей VRF                              |
| `net.ipv4.conf.all.src_valid_mark`    | `1`      | требуется для PBR на базе fwmark                                     |
| `net.ipv4.tcp_l3mdev_accept`          | `1`      | cross-VRF socket lookup и для v4, и для v6 (несмотря на префикс `ipv4`) |
| `net.ipv4.udp_l3mdev_accept`          | `1`      | то же — обе семьи                                                    |
| `net.ipv4.raw_l3mdev_accept`          | `1`      | cross-VRF ICMP                                                       |
| `net.ipv4.ip_forward`                 | `1`      | узел — маршрутизатор                                                 |
| `net.ipv6.conf.all.forwarding`        | `1`      | то же для v6                                                         |

Справка: [Documentation/networking/ip-sysctl.rst](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.rst).

## `vrf-vpn`

Источник: [`roles/vrf-vpn/`](../../roles/vrf-vpn/)

Что делает:

1. Грузит модуль ядра `vrf` (one-shot на apply + module-load.d для boot).
2. Пишет `infra-loopback.service` — присваивает BGP router-id `/32` и (если `secure_dns_enabled`) DNS service-IP'ы на `lo`.
3. Пишет `vrf-vpn.service` — создаёт master `vrf-vpn`, unreachable-backstop в table 10, return-маршруты в main, sibling-pool return-маршруты в main, cross-VRF default в table 10 и (если `secure_dns_enabled`) v6 ip-rule для DNS.
4. Добавляет `vrf-vpn` и `vpns+` в firewalld-зону `trusted`.

Цепочка `ExecStart` в `vrf-vpn.service` (шаблонизирована — смотри [`templates/vrf-vpn.service.j2`](../../roles/vrf-vpn/templates/vrf-vpn.service.j2)):

```ini
ExecStart=/usr/sbin/ip link add vrf-vpn type vrf table 10
ExecStart=/usr/sbin/ip link set vrf-vpn up
ExecStart=/usr/sbin/ip -4 route replace unreachable default table 10 metric 4278198272
ExecStart=/usr/sbin/ip -6 route replace unreachable default table 10 metric 4278198272
ExecStart=/usr/sbin/ip -4 route replace 100.64.X.0/24 dev vrf-vpn          # собственный клиентский пул — return в main FIB
ExecStart=/usr/sbin/ip -6 route replace fdf3:bb42:9fc6:X::/64 dev vrf-vpn  # то же v6
ExecStart=/usr/sbin/ip -4 route replace 100.64.Y.0/24 via 169.254.255.Z dev wgN  # sibling-pool return
ExecStart=/usr/sbin/ip -6 route replace fdf3:bb42:9fc6:Y::/64 via fe80::N dev wgN
ExecStart=/usr/sbin/ip -4 route replace 172.20.0.0/24 dev podman4 table 10        # только если secure_dns
ExecStart=/usr/sbin/ip -6 rule add iif "vpns+" to fdf3:…:Y::53/128 lookup main pref 900   # только если secure_dns
ExecStart=/usr/sbin/ip -4 route replace default via <gw> dev <iface> table 10
ExecStart=/usr/sbin/ip -6 route replace default via <gw> dev <iface> table 10
```

Справка: [документация Linux VRF](https://docs.kernel.org/networking/vrf.html).

## `nft-vpn`

Источник: [`roles/nft-vpn/`](../../roles/nft-vpn/)

Что делает:

- Всегда: разворачивает `/etc/nft.d/mss_clamp.nft` + one-shot `nft-mss-clamp.service`, который грузит его на boot.
- Если `secure_dns_enabled`: разворачивает `/etc/nft.d/vpn_dnat.nft` + соответствующий сервис.

`mss_clamp.nft` зажимает TCP MSS на inter-site WG-интерфейсе (шаблонизированный `{{ wg_sibling_iface }}`) — нужно потому, что RU-сайты в частности дропают ICMP-PTB и иначе блэкхолили бы большие пакеты.

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

`vpn_dnat.nft` — workaround для cross-VRF DNAT-разрыва netavark:

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

Почему priority `-101` (на единицу раньше netavark'овского `-100`): netavark гейтит DNAT на `fib daddr type local`, что ложно для service-IP с точки зрения `vrf-vpn`. Хук раньше с явным iif пропускает пакет до Pi-hole.

Справка: [nftables wiki — atomic rule replacement](https://wiki.nftables.org/wiki-nftables/index.php/Atomic_rule_replacement); [netavark nft DNAT source](https://github.com/containers/netavark/blob/main/src/firewall/nft.rs).

## `secure-dns`

Источник: [`roles/secure-dns/`](../../roles/secure-dns/)

Пропускается, когда `secure_dns_enabled` false (по умолчанию для country-exit узлов).

Что делает:

- Ставит `podman` + `podman-compose`.
- Пишет `compose.yaml` в `/opt/project/repositories/secure-dns-vpn/` с Pi-hole, привязанным к host-IP `<v4 service-IP>`, и приватным мостом `172.20.0.0/24`.
- Пишет `.env` с секретами из vault.
- Пишет два systemd-юнита — `dns-v6-proxy.service` и `dns-v6-proxy-tcp.service`, которые запускают `socat` на v6 service-IP с форвардом в `127.0.0.1:53` (потому что netavark не реализует IPv6 DNAT).
- Добавляет мост `podman4` в firewalld-зону `trusted`.

Сводка DNS-пути:

```text
client → vpns0 → nft DNAT (vpn_dnat) → podman4 bridge → Pi-hole 172.20.0.20:53
                                                              ↓
                                                       dnscrypt-proxy 172.20.0.10:5053
                                                              ↓
                                                       Cloudflare / Quad9 через DoH
```

Справки: [Pi-hole](https://docs.pi-hole.net/), [dnscrypt-proxy](https://dnscrypt.info/).

## `ocserv`

Источник: [`roles/ocserv/`](../../roles/ocserv/)

Что делает:

- Ставит `ocserv` (`dnf install ocserv`).
- Пишет `/etc/ocserv/connect-vrf.sh` — хук connect-script, который загоняет per-client туннель `vpns*` в `vrf-vpn` и устанавливает v6 `/128` клиента в **table 10**.
- Рендерит `/etc/ocserv/ocserv.conf` из `templates/ocserv.conf.j2`, используя `ocserv_vhosts` + `push_dns_resolvers` + per-host пулы.
- Валидирует конфиг через `ocserv -t` перед записью.
- Notify-ит handler `occtl reload` (graceful — активные сессии не сбрасывает).

Тело `connect-vrf.sh` короткое, но критично-нагруженное:

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

Почему `table 10` явно указан на v6 add: ocserv добавляет маршрут через `SIOCADDRT`, который по умолчанию ставит в `RT_TABLE_MAIN` независимо от VRF master'а slave-интерфейса — а v6 reply-путь нужен в table 10 vrf-vpn.

### Compat-директивы, которые везёт шаблон

Шаблон по умолчанию везёт эти legacy-compat директивы включёнными, держим их потому, что в продакшен-клиентах всё ещё попадаются старые устройства, производные от AnyConnect:

| Директива                       | По умолчанию | Когда выключать                                                         |
|---------------------------------|--------------|-------------------------------------------------------------------------|
| `cisco-client-compat = true`    | on           | Все клиенты — OpenConnect ≥ 8.0; AnyConnect Mobile ≥ 4.x                |
| `cisco-svc-client-compat = true`| on           | То же — deprecated в новых сборках ocserv; смотри warnings `ocserv -t`  |
| `dtls-legacy = true`            | on           | Все клиенты договариваются о DTLS 1.2 (`gnutls-cli --priority=NORMAL ...`) |
| `compression = false`           | off          | Держать выключенным — DTLS уже сжимает; риски класса CRIME              |
| `try-mtu-discovery = true`      | on           | Поставить `false` + запинить `mtu`, когда PMTUD через границу VRF ведёт себя плохо |

Аудит достижимых клиентов — через `occtl show users`; колонка `dtls-cipher` показывает, каким сессиям всё ещё нужны DTLS-legacy пути.

Справка: [мануал `ocserv`](https://ocserv.openconnect-vpn.net/ocserv.8.html), [`ocserv` sample config](https://gitlab.com/openconnect/ocserv/-/raw/master/doc/sample.config).

## `georoute`

Источник: [`roles/georoute/`](../../roles/georoute/)

Пропускается, если `country` не определён в host vars.

Что делает:

1. Собирает Go-бинарь `georoute` на контроллере (`delegate_to: localhost`) и копирует его в `/usr/local/bin/georoute` на target.
2. Пишет `/etc/nft.d/pbr.nft`, объявляющий country set + prerouting/output-цепочки, которые ставят country `fwmark`.
3. Пишет `nft-pbr.service`, чтобы грузить `pbr.nft` на boot.
4. Добавляет `ip rule fwmark <country.fwmark> lookup <country.pbr_table>` для v4 и v6.
5. Ставит per-country PBR-таблице default на локальный uplink узла.
6. Пишет `/etc/georoute/<cc>.env` и systemd-шаблоны `georoute@.service` + `georoute@.timer`.
7. Включает `georoute@<cc>.timer`.

Каденс таймера: `OnBootSec=5min`, `OnUnitActiveSec=12h`, `RandomizedDelaySec=30min`.

Сам бинарь `georoute` документирован отдельно в [09-georoute.md](09-georoute.md).

## Что читать дальше

- [Развёртывание](06-deployment.md) — первое применение, частые флаги.
- [Добавление country exit](07-country-exit-bootstrap.md) — production-чеклист ввода.
