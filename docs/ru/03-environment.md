# 03 — Окружение

## Host OS

**Oracle Linux 10 (OL10) с UEK Release 8** (kernel `6.12.x-uekR8`).

- OL10 бинарно совместим с RHEL10 и разделяет с ним вселенную пакетов (RPM через `dnf`). Справка: [Oracle Linux 10 release notes](https://docs.oracle.com/en/operating-systems/oracle-linux/10/relnotes10/).
- UEK R8 везёт ядро `6.12-LTS`, которое нужно для:
  - First-class VRF (`l3mdev`) data path. Справка: [Linux VRF doc](https://docs.kernel.org/networking/vrf.html).
  - `nft` hook `route` на output (используется `pbr.nft`).
  - `tcp_l3mdev_accept` / `udp_l3mdev_accept`, покрывающие и v4, и v6.

Любой современный EL10-совместимый дистрибутив (Rocky 10, Alma 10, RHEL 10) работает так же; playbook не зависит от Oracle-specific пакетов.

## Архитектура

Только `x86_64` — модули `firewalld` и `nftables` собраны и протестированы на этой арке. Ansible-инвентарь не гейтит на `ansible_architecture`, но в production пробовался только `amd64`.

## Требуемые модули ядра

| Модуль        | Зачем                                                                   |
|---------------|-------------------------------------------------------------------------|
| `vrf`         | Master-устройство `vrf-vpn` — приходит из `kernel-uek-modules-extra`.   |
| `nf_tables`   | Все `nft`-правила используют семейство `inet` — встроено в базовое ядро. |
| `nf_nat`      | DNAT/SNAT от `firewalld` и `vpn_dnat` — встроено.                        |
| `xt_TCPMSS`   | MSS clamp — приходит из `kernel-uek-modules-extra-netfilter`.           |

Персистентность: `/etc/modules-load.d/vrf.conf` (разворачивается ролью `vrf-vpn`) пишет `vrf\n`, чтобы `vrf` грузился на каждой загрузке.

## Список пакетов (top-level `base_packages`)

Определён в [`inventory/group_vars/all.yml`](../../inventory/group_vars/all.yml). Резолвится через `dnf install`:

```yaml
base_packages:
  - nftables
  - firewalld
  - iproute
  - iproute-tc
  - socat
  - bind-utils
  - tcpdump
  - conntrack-tools
  - jq
  - git
  - vim
  - bash-completion
  - chrony
  - kernel-uek-modules-extra
```

Per-role пакеты добавляются по требованию:

| Роль         | Дополнительные пакеты                                   | Зачем                                                       |
|--------------|---------------------------------------------------------|-------------------------------------------------------------|
| `ocserv`     | `ocserv`                                                | OpenConnect SSL VPN-сервер. [Страница проекта](https://ocserv.gitlab.io/www/index.html). |
| `secure-dns` | `podman`, `podman-compose`, `python3-pip`               | Стек Pi-hole + dnscrypt. Pi-hole-образ из `docker.io/pihole/pihole:latest`. |
| `georoute`   | ничего на target — Go-бинарь кросс-собирается на контроллере | Избегаем тащить Go-тулчейн на каждый exit-узел.        |

## Почему именно эти

| Инструмент                  | Почему именно он, а не альтернатива                                                    |
|-----------------------------|----------------------------------------------------------------------------------------|
| `ocserv` (а не `OpenVPN`)   | CSTP+DTLS, быстрый handshake, хорошо живёт в корпоративных сетях; долгоживущий проект. |
| `FRR` (а не `BIRD`)         | Шире развёрнут в enterprise; `frr-reload.py` позволяет инкрементальный hot reload.    |
| `nftables` (а не `iptables`) | Атомарные транзакции; единый ruleset на v4+v6 через семейство `inet`.                |
| `firewalld` (а не сырой `nft`) | Зональная абстракция; лёгкий `--add-interface` для новых туннелей.                |
| `WireGuard` (а не `IPsec`)  | Меньшая поверхность атаки ядра; детерминированный обмен ключами; конфиг одним файлом.  |
| `podman` (а не `docker`)    | Rootless-опция; без демона; systemd-дружественный; предустановлен в EL10.              |
| `Pi-hole` (опционально)     | Готовый UI фильтрации; заменим на `unbound` для чистой рекурсии.                       |
| `Ansible` (а не `Saltstack`) | Push-only; SSH-only; никаких лишних агентов на таргетах; только YAML.                |

## Бюджет дискового пространства

```text
/                       ~5 GB     базовый OL10 + пакеты
/var/lib/containers     ~1 GB     pihole-образ + персистентные тома (если secure-dns)
/etc                    ~20 MB
/var/log                растёт — journald compaction по умолчанию в /etc/systemd/journald.conf
```

Корневой раздел `30 GB` комфортен.

## Сетевые требования на узел

| Направление | Порт           | Назначение                                             |
|-------------|----------------|--------------------------------------------------------|
| inbound     | TCP/22         | SSH для Ansible                                        |
| inbound     | TCP/443        | ocserv CSTP control + data fallback                    |
| inbound     | UDP/443        | ocserv DTLS — опционально (некоторые DPI это режут)    |
| inbound     | UDP/<wg-port>  | WireGuard между соседями (по умолчанию 31518; per-host настраиваемо) |
| outbound    | TCP/443        | Letsencrypt + RIPE Stat для georoute                   |
| outbound    | UDP/53         | DNS-резолюция (для ACME challenge)                     |

Ansible-роли сами эти порты не открывают — они предполагают, что зоны `firewalld` преднастроены (смотри [`roles/common`](../../roles/common/tasks/main.yml)).

## Время

`chrony` ставится ролью `common`, но его `chrony.conf` **не** управляется. Рекомендуемая конфигурация: пинуй пулы поближе к географии. Пример для RU:

```ini
pool 0.ru.pool.ntp.org iburst
pool time.cloudflare.com iburst
```

Дрейф NTP ломает ротацию ключей DTLS и валидацию TLS-сертов — держи плотно.

## SELinux / AppArmor

Оба **выключены** на production-узлах. Ansible-playbook не трогает состояние SELinux. Если включаешь `enforcing`:

- `ocserv` требует `setsebool ocserv_use_userspace_libraries on` и метку для `/etc/ocserv/connect-vrf.sh`.
- `podman` на EL10 обычно работает с дефолтной политикой.
- `nft` и `firewalld` SELinux не трогает.

## Что читать дальше

- [Инвентарь](04-inventory.md) — моделирование узла.
- [Развёртывание](06-deployment.md) — первое применение.
