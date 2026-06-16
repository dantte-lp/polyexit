# 07 — Добавление country exit

End-to-end ввод нового country-exit узла, иллюстрируется на примере Узбекистана (`dev-05`, ISO `UZ`, ISO-numeric `860`).

## Предусловия

| Пункт                 | Требуемое значение                                                             |
|-----------------------|--------------------------------------------------------------------------------|
| VPS в стране          | статический публичный IPv4; IPv6 желателен                                     |
| Root SSH-доступ       | только по ключу; pubkey контроллера добавлен в `/root/.ssh/authorized_keys`    |
| Провайдер не блокирует | TCP/443 (обязательно) и UDP/443 (желательно), плюс выбранный WG-порт          |
| 30 GB диска           | корневой раздел                                                                |
| 1 GB RAM              | минимум; 2 GB комфортно                                                        |

## Предварительное выделение идентификаторов

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
WG iface name:  wg103                  # на dev-03 = wg<sibling_site_octet * 100>
                wg0                    # на dev-05 — единственный туннель в сторону dev-03
WG /31 slot:    169.254.255.6/7        # пара (3,5) — смотри схему адресации
WG v6 link-local: fe80::3 (сторона dev-03), fe80::5 (сторона dev-05)
```

`0x205` / table 105 / community `64512:2860` производятся из `site_octet` и `iso_numeric` — формулы в [Инвентаре](04-inventory.md).

## Шаг 1 — выдели VPS

Что угодно, что даёт root SSH на Oracle Linux 10 / Rocky 10 / Alma 10. Справка: [Oracle Linux 10 install guide](https://docs.oracle.com/en/operating-systems/oracle-linux/10/install/).

После первого логина:

```bash
dnf install -y python3            # требуется Ansible'у для fallback `raw` к play
```

Это всё, что должно произойти out-of-band — остальная сборка в playbook'е.

## Шаг 2 — инвентарь

Правь [`inventory/hosts.yml`](../../inventory/hosts.yml):

```yaml
exits-country-local:
  hosts:
    dev-05:
      ansible_host: 203.0.113.5        # ← реальный IP UZ VPS
      ansible_user: root
      ansible_port: 22
      ansible_connection: ssh
```

Правь [`inventory/host_vars/dev-05.yml`](../../inventory/host_vars/dev-05.yml) — большинство полей уже заполнено UZ-плейсхолдерами. Обнови четыре строки `TBD`:

```yaml
public_v4: 203.0.113.5
public_v6: ""                          # коммерческий v6 в UZ слабый — оставь пустым
host_v4_uplink: ens1                   # проверь через `ip route show default`
host_v4_gw: 203.0.113.1                # GW провайдера
```

## Шаг 3 — контроллер достукивается

```bash
ansible-playbook playbooks/site.yml --limit dev-05 -m ping
# ожидаемо:
# dev-05 | SUCCESS => {"ansible_facts": {...}, "ping": "pong"}
```

Если падает — чини SSH, прежде чем двигаться дальше.

## Шаг 4 — WireGuard-туннель до dev-03 (sibling-of-record)

```bash
# на dev-05
dnf install -y wireguard-tools
mkdir -p /etc/wireguard
umask 077
wg genkey | tee /etc/wireguard/wg0.key | wg pubkey > /etc/wireguard/wg0.pub
wg genpsk > /etc/wireguard/wg0.psk      # общий с dev-03

# на dev-03
wg genkey | tee /etc/wireguard/wg103.key | wg pubkey > /etc/wireguard/wg103.pub
# используй ОДИН psk на обеих сторонах
```

Настрой обе стороны — `/etc/wireguard/wg0.conf` на dev-05:

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
Endpoint = 192.0.2.3:31518
PersistentKeepalive = 25
```

`/etc/wireguard/wg103.conf` на dev-03 — зеркальная копия. Подними обе:

```bash
systemctl enable --now wg-quick@wg0    # на dev-05
systemctl enable --now wg-quick@wg103  # на dev-03
ping -c2 169.254.255.6                 # с dev-05 — должно отвечать
```

## Шаг 5 — секции FRR-соседей

Добавь в `/etc/frr/frr.conf` на dev-03 в блок `router bgp 64512`:

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

Плюс соответствующие route-maps с `match community CL-CC-UZ` (community `64512:2860`) и `set local-preference 200` на inbound, `set community 64512:2860 additive` на анонсируемых страновых префиксах.

`frr.conf` для dev-05 будет шаблонизирован будущей FRR Ansible-ролью; пока скопируй структуру с dev-04 и подстрой ASN / IP-соседей.

Тест:

```bash
vtysh -c 'show bgp summary'           # peer должен быть Established
```

## Шаг 6 — применить playbook

```bash
ansible-playbook playbooks/site.yml --limit dev-05
```

Ожидаемые эффекты на dev-05:

- роли `common`, `vrf-vpn`, `nft-vpn`, `ocserv`, `georoute` применены
- `/usr/local/bin/georoute` развёрнут
- `pbr.nft` загружен с пустыми sets `uz_v4` / `uz_v6`
- `georoute@uz.timer` включён (сработает через 5 мин после boot или на следующем прогоне)

## Шаг 7 — прогони georoute один раз руками (не жди 5 мин)

```bash
ssh root@dev-05 systemctl start georoute@uz.service
ssh root@dev-05 journalctl -u georoute@uz.service -n 10 --no-pager
# ожидаемая последняя строка:
#   georoute[UZ] ... nft sets updated (uz_v4=162 uz_v6=48)
#   georoute[UZ] ... frr-reload completed
```

## Шаг 8 — провалидируй BGP-пропагацию на dev-03

```bash
# на dev-03
vtysh -c 'show bgp summary' | grep dev-05      # peer Established, PfxRcd ≈ 162+48
vtysh -c 'show bgp ipv4 unicast community 64512:2860' | head
ip -4 route show table 10 | grep wg103 | head  # UZ-префиксы через wg103
```

## Шаг 9 — end-to-end тест с реального VPN-клиента

Подключись к `news.infra4.dev` или `docs.infra4.dev` (Cloudflare multi-A round-robin на dev-03 / dev-04 / dev-05).

```bash
# с VPN-клиента
curl -4 ifconfig.io                          # жди Finland IP (default exit dev-03)
curl -4 -H "Host: ifconfig.io" 5.255.255.77  # этот RU IP — жди exit IP Москвы
curl -4 -H "Host: ifconfig.io" 203.0.113.1   # UZ IP — жди exit IP Ташкента
```

`ifconfig.io` возвращает source IP — напрямую читаешь, какой exit взял трафик.

## Шаг 10 — алерт на падение BGP

Вне scope этого playbook'а, но рекомендуется:

```bash
# проверяй каждые 5 мин — пейджи, если число country-префиксов падает >50%
vtysh -c 'show bgp ipv4 unicast community 64512:2860' | grep -c '^[*>]'
```

Пуш число в свою систему мониторинга (Prometheus, Zabbix, Healthchecks — на твой выбор).

## Что читать дальше

- [Управление пользователями](08-user-management.md) — сделай новый exit пригодным.
- [Troubleshooting](10-troubleshooting.md) — когда один из шагов выше ломается.
