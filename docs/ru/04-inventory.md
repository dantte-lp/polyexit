# 04 — Инвентарь

## Файлы

```text
inventory/
├── hosts.yml                    # принадлежность к группам и узлам
├── group_vars/
│   ├── all.yml                  # общее для всех узлов
│   └── vault.yml.example        # шаблон для секретов (зашифруй перед коммитом)
└── host_vars/
    ├── dev-03.yml               # FI world-default
    ├── dev-04.yml               # RU country-exit
    └── dev-05.yml               # UZ country-exit (заглушка)
```

## `hosts.yml` — иерархия групп

```yaml
all:
  children:
    exits:
      children:
        exits-world:
          hosts:
            dev-03:
              ansible_host: 82.26.171.163
              ansible_connection: local
        exits-country-local:
          hosts:
            dev-04:
              ansible_host: 91.218.113.232
              ansible_connection: ssh
            dev-05:
              ansible_host: "{{ public_v4 }}"
              ansible_connection: ssh
```

- **`exits-world`** — узлы, у которых VPN-клиенты уходят через собственный uplink узла.
- **`exits-country-local`** — узлы, у которых VPN-клиенты уходят через собственный uplink *только* для префиксов своей страны; весь остальной трафик транзитится через WireGuard на world-default соседа.

Узел может быть только в одной из этих подгрупп.

## `group_vars/all.yml`

Переменные, общие для всего флота. Примеры:

```yaml
bgp_asn: 64512                          # legacy ASN для dev-03 (совместимость с Mikrotik)
ula_prefix: "fdf3:bb42:9fc6::/48"       # RFC 4193 ULA — нарезанный /64-на-узел
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

## `host_vars/dev-NN.yml` — модель узла

У каждого узла есть один такой. Форма идентична; различаются только значения. Обязательные поля:

```yaml
# Identity + reachability
public_v4: 91.218.113.232          # собственный публичный v4 узла
public_v6: "2a03:e2c0:5acd::2"     # собственный публичный v6 узла (или "" если v4-only)
site: ru                           # короткий слаг
role: country-exit                 # world-default | country-exit
site_octet: 4                      # 1-байтный детерминированный ключ — от него производится много переменных

# Собственные uplinks узла
host_v4_uplink: ens1
host_v4_gw: 91.218.113.129
host_v6_uplink: sit1

# VRF default exit — куда VPN-клиенты в vrf-vpn уходят для non-country трафика
v4_uplink_iface: wg0               # country-exit: wg к world-default; world-default: ens1
v4_uplink_gw: 169.254.255.0
v6_uplink_iface: wg0
v6_uplink_gw: "fe80::3"
wg_sibling_iface: wg0              # имя inter-site WG-интерфейса на ЭТОМ узле

# Клиентские пулы — детерминированы из site_octet
v4_pool_cidr: 100.64.4.0/24
v4_pool_netmask: 255.255.255.0
v4_pool_network: 100.64.4.0
v4_pool_gw: 100.64.4.1
v6_pool_cidr: "fdf3:bb42:9fc6:4::/64"
v6_pool_gw: "fdf3:bb42:9fc6:4::1"

# Pi-hole service-IPs (используются только если secure_dns_enabled=true)
dns_service_v4: 100.64.4.53
dns_service_v6: "fdf3:bb42:9fc6:4::53"
secure_dns_enabled: false
push_dns_resolvers:
  - "9.9.9.9"
  - "2606:4700:4700::1111"

# BGP
bgp_router_id_v4: 10.255.0.4
bgp_router_id_v6: "fdf3:bb42:9fc6:ff00::4"

# Соседний сайт для transit-return маршрутов
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

# Country-блок — управляет ролью georoute
country:
  iso2: RU
  iso_numeric: 643
  bgp_community: "64512:201"          # legacy для RU; новые страны используют "64512:2<iso_numeric>"
  route_map: MARK-RU-EXIT
  nft_set_prefix: ru
  feed_url: "https://stat.ripe.net/data/country-resource-list/data.json?resource=RU&v4_format=prefix"
  fwmark: "0x201"
  pbr_table: 100
```

## Производные имена

Многие host-level переменные детерминированы из `site_octet`:

| Производная переменная  | Формула                                       | Пример (`site_octet: 4`)              |
|-------------------------|-----------------------------------------------|---------------------------------------|
| `v4_pool_cidr`          | `100.64.<octet>.0/24`                         | `100.64.4.0/24`                       |
| `v6_pool_cidr`          | `fdf3:bb42:9fc6:<octet>::/64`                 | `fdf3:bb42:9fc6:4::/64`               |
| `bgp_router_id_v4`      | `10.255.0.<octet>`                            | `10.255.0.4`                          |
| `bgp_router_id_v6`      | `fdf3:bb42:9fc6:ff00::<octet>`                | `fdf3:bb42:9fc6:ff00::4`              |
| `country.fwmark`        | `0x200 | site_octet` (рекомендуется для новых) | новые: `0x205`; legacy RU остаётся `0x201` |
| `country.pbr_table`     | `100 + site_octet` (рекомендуется для новых)  | новые: `105`; legacy RU остаётся `100` |

> Legacy-узлы (dev-04) сохраняют свои оригинальные `0x201` / table 100 для обратной совместимости. Новые узлы должны следовать формулам.

## Секреты — `vault.yml`

Шифруй через `ansible-vault`:

```bash
cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml
$EDITOR inventory/group_vars/vault.yml
ansible-vault encrypt inventory/group_vars/vault.yml
echo 'YOUR_VAULT_PASSWORD' > ~/.ansible_vault_pass
chmod 600 ~/.ansible_vault_pass
# раскомментируй строку `vault_password_file` в ansible.cfg
```

Распознаваемые переменные внутри `vault.yml`:

```yaml
vault_cf_api_token: "..."             # Cloudflare DNS API token (для deploy-hook серта)
vault_pihole_web_password: "..."      # пароль веб-админки Pi-hole (роль secure-dns)
vault_bgp_md5: "..."                  # eBGP MD5 password (FRR — пока не шаблонизировано)
```

[`group_vars/all.yml`](../../inventory/group_vars/all.yml) ссылается на них как `vault_*`, чтобы было очевидно, откуда они пришли. Справочная документация: [ansible-vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

## Добавление нового узла

1. Выдели `site_octet`. Конвенция: 3 = FI, 4 = RU, 5 = UZ, 6 = KZ и т. д.
2. Создай `inventory/host_vars/dev-NN.yml` по модели выше.
3. Добавь в `hosts.yml` под правильную группу (`exits-world` или `exits-country-local`).
4. Прогони [bootstrap-чеклист](07-country-exit-bootstrap.md).

## Что читать дальше

- [Роли](05-roles.md) — что делает каждая роль и что трогает.
- [Развёртывание](06-deployment.md) — первое применение.
