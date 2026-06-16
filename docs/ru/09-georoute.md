# 09 — georoute

Один Go-бинарь, который:

1. Тянет список префиксов страны из [RIPE Stat country-resource-list](https://stat.ripe.net/docs/data_api).
2. Агрегирует смежные / перекрывающиеся префиксы.
3. Атомарно заменяет содержимое `nft inet pbr <cc>_v4` и `<cc>_v6` через `nft -f -`.
4. Сплайсит строки `network X/Y route-map MARK-<CC>-EXIT` в `/etc/frr/frr.conf` между marker-комментариями.
5. Прогоняет `frr-reload.py --reload`, только когда рендеренный блок изменился.

Источник: [github.com/dantte-lp/georoute](https://github.com/dantte-lp/georoute) (отдельный публичный репозиторий).

## CLI-флаги

```text
-country string         ISO-3166 alpha-2 код страны (RU, UZ, KZ, …) (default "RU")
-route-map string       имя FRR route-map (default MARK-<CC>-EXIT)
-nft-set-v4 string      имя v4 set nftables (default <cc>_v4)
-nft-set-v6 string      имя v6 set nftables (default <cc>_v6)
-marker-prefix string   префикс marker-комментария (default <CC>-FEED)
-feed-url string        RIPE Stat URL (default country-resource-list для <cc>)
-lock-file string       путь эксклюзивного flock (default /run/georoute-<cc>.lock)
-frr-conf string        путь к конфигу FRR (default /etc/frr/frr.conf)
-reload                 запускать frr-reload при изменении (default true)
-nft                    атомарно заменять nft set inet pbr {<cc>_v4,<cc>_v6} (default true)
-dry-run                напечатать summary без записи
-force                  форс-запись, даже если без изменений
```

Дефолты пяти флагов `-route-map / -nft-set-* / -marker-prefix / -feed-url / -lock-file` все производятся из `-country`, так что для любой страны достаточно одного аргумента:

```bash
georoute --country=UZ --dry-run
georoute --country=KZ
```

## Конфигурация — `/etc/georoute/<cc>.env`

Ansible-роль `georoute` пишет по одному env-файлу на каждую страну, которой владеет узел. Пример (`/etc/georoute/ru.env`):

```ini
GEOROUTE_COUNTRY=RU
GEOROUTE_FEED_URL=https://stat.ripe.net/data/country-resource-list/data.json?resource=RU&v4_format=prefix
GEOROUTE_ROUTE_MAP=MARK-RU-EXIT
GEOROUTE_NFT_SET_V4=ru_v4
GEOROUTE_NFT_SET_V6=ru_v6
GEOROUTE_MARKER_PREFIX=RU-FEED
GEOROUTE_FRR_CONF=/etc/frr/frr.conf
```

Шаблон `georoute@.service` ссылается на него через `EnvironmentFile=`.

## Systemd-юниты

```text
/etc/systemd/system/georoute@.service     # template service
/etc/systemd/system/georoute@.timer       # template timer
```

Включай на каждую страну:

```bash
systemctl enable --now georoute@ru.timer
systemctl list-timers georoute@*.timer
```

Каденс: `OnBootSec=5min`, `OnUnitActiveSec=12h`, `RandomizedDelaySec=30min`. RIPE Stat терпим к такой частоте.

## Свойства безопасности

| Свойство                                           | Механизм                                                                                  |
|----------------------------------------------------|-------------------------------------------------------------------------------------------|
| Два параллельных прогона не могут гонять `frr.conf` | `flock` на `/run/georoute-<cc>.lock`                                                     |
| Запись `frr.conf` атомарна                         | `os.CreateTemp` + `rename` в той же директории (без общего суффикса `.new`)               |
| Замена `nft` set атомарна                          | один вызов `nft -f -` — ядро применяет как одну netlink-транзакцию                        |
| Враждебный JSON не может OOM-нуть узел             | `io.LimitReader(_, 32 MiB)` на response body                                              |
| Транзиентный RIPE 503/429 не пропускает 12-часовой цикл | 3 попытки с экспоненциальным backoff                                                  |
| `frr-reload.py` не может превысить свой time budget | выделенный 3-минутный child context (parent ctx — 5 мин суммарно)                         |
| Враждебный env от misbehaving caller'а не протекает | `cmd.Env = []string{"PATH=/usr/sbin:/usr/bin:/sbin:/bin"}`                                |
| Идемпотентность                                    | SHA-256 хеш рендеренного блока — `frr.conf` переписывается только при diff (если не `--force`) |

Справки: [nftables wiki — atomic rule replacement](https://wiki.nftables.org/wiki-nftables/index.php/Atomic_rule_replacement), [Go `os.CreateTemp`](https://pkg.go.dev/os#CreateTemp), [`flock(2)`](https://man7.org/linux/man-pages/man2/flock.2.html).

## Команды оператора

```bash
# Dry-run RU — печатает summary, без записей
georoute --country=RU --dry-run

# Force-запись, даже если без изменений (post-recovery)
georoute --country=RU --force

# Проверить, что конкретное назначение покрыто set'ом
nft get element inet pbr ru_v4 { 77.88.55.88 }

# Время последнего прогона
journalctl -u georoute@ru.service -n 10 --no-pager

# Размер set
nft list set inet pbr ru_v4 | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[0-9]+' | wc -l
```

## Что пишется в `frr.conf`

Внутри `router bgp <ASN>`, между маркерами:

```text
 address-family ipv4 unicast
  network 100.64.4.0/24
  network 10.255.0.4/32
  ! BEGIN-RU-FEED-V4
  network 2.56.24.0/22 route-map MARK-RU-EXIT
  network 2.56.88.0/22 route-map MARK-RU-EXIT
  network 2.56.180.0/22 route-map MARK-RU-EXIT
  …  (~8 600 строк для RU)
  ! END-RU-FEED-V4
 exit-address-family
```

Маркеры **обязаны** быть добавлены в `frr.conf` руками один раз — `georoute` пишет только между ними, никогда не вставляет их сам. Начальный bootstrap:

```text
  ! BEGIN-RU-FEED-V4
  ! END-RU-FEED-V4
```

То же для `-V6`.

## Когда фид меняет форму

RIPE Stat периодически добавляет или убирает префиксы из delegated-списка страны. `georoute` перепрогоняется каждые 12 ч, чтобы автоматически подхватить эти изменения. Exit IP для затронутых назначений меняется на границе следующего прогона; существующие TCP-сессии к этим назначениям могут порваться и потребовать переподключения.

## Что читать дальше

- [Troubleshooting](10-troubleshooting.md) — что делать, когда гео-маршрутизация ведёт себя странно.
- [References](11-references.md) — первичные доки по `nft`, `FRR`, `RIPE Stat`.
