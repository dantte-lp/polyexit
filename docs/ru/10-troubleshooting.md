# 10 — Troubleshooting

## Методология

> Server-local тесты **не** валидируют туннельные пути. Всегда тестируй с реального VPN-клиента (или симулируй его в netns).

Правило большого пальца:

```text
СИМПТОМ: "фича X сломана для VPN-клиента"
НЕВЕРНО: `ping`/`curl` из шелла узла — использует default-VRF маршрутизацию
ВЕРНО  : тестируй с подключённого клиента (или `ip vrf exec vrf-vpn …` с правильным src)
```

## Матрица симптомов

| Симптом                                              | Первая проверка                                                          | Вероятная причина                                                             |
|------------------------------------------------------|--------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| Клиент подключается, но трафик не выходит            | `nft list table inet firewalld | grep masquerade`                        | Неправильная зона oif — `vpns* → wg*` это `trusted→trusted`, masquerade не срабатывает |
| RU-префикс выходит через FI                          | `nft get element inet pbr ru_v4 { <dst> }`                               | Set `ru_v4` пустой — прогони `systemctl start georoute@ru.service`            |
| Ответ клиенту соседа уходит в чёрную дыру            | `ip route show <sibling pool>`                                           | Маршрут sibling pool пропал — фильтр `route-map BGP-MAIN-FIB`; гарантируй static |
| v6 ping6 к назначению таймаутит (NAT'ный путь)       | `ip -6 rule | grep "lookup 50"`                                          | `from <own-ipv6> lookup 50` матчит и ответ тоже; добавь return-маршрут в table 50 |
| Pi-hole DNS таймаутит (UDP/53)                       | `nft list table inet vpn_dnat | grep counter`                            | Counter == 0: DNAT не матчится — проверь iif-правило (`vpns*` и `vrf-vpn`)    |
| `vpns0` виден в `occtl`, но клиент получает только gateway-RX | `journalctl -t ocserv-vrf -n 10`                                | connect-script не запускается — `getsebool ocserv_use_userspace_libraries`    |
| WG handshake никогда не завершается                  | `wg show wg0 | grep handshake`                                            | IP peer `endpoint` поменялся; PSK не совпадает; UDP-порт блокирован           |
| BGP-peer хлопает                                     | `vtysh -c 'show bgp neighbor <ip>'`                                      | MD5 mismatch; опечатка в ASN; `address-family` не активирована на соседе      |

## Live-диагностика

### 1. Где умирает пакет?

```bash
# добавь one-shot trace для одного 5-tuple
nft 'add rule inet filter input ip daddr 1.1.1.1 meta nftrace set 1'
nft monitor trace                    # в другом шелле, пока воспроизводишь

# или шире — каждая причина drop
perf trace -e skb:kfree_skb
```

Справки: [nftables ruleset debug](https://wiki.nftables.org/wiki-nftables/index.php/Ruleset_debug/tracing); [SKB drop reasons enum](https://dxuuu.xyz/dropreason.html).

### 2. Cross-VRF routing-решение

```bash
# что сделало бы ядро для пакета клиент → 8.8.8.8?
ip route get 8.8.8.8 from 100.64.4.50 iif vrf-vpn

# v6-эквивалент
ip -6 route get 2606:4700:4700::1111 from fdf3:bb42:9fc6:4::dead iif vrf-vpn
```

Если назначение должно идти через wg0, а ответ говорит `r64stk` — твой default в table 10 неправильный — перепрогони роль `vrf-vpn`.

### 3. Префикс в set'е?

```bash
nft get element inet pbr ru_v4 { 77.88.55.88 }
# ожидаемо:
# table inet pbr { set ru_v4 { type ipv4_addr; flags interval; elements = { 77.88.0.0/18 } } }
```

Если ошибка `No such file or directory` — IP не покрыт ни одним элементом set. Либо страновой фид legitimately его не включает, либо `ru_v4` был очищен (перепрогони `georoute --force`).

### 4. georoute запускался недавно?

```bash
systemctl list-timers georoute@*.timer
journalctl -u georoute@ru.service -n 20 --no-pager
```

Ищи `nft sets updated (ru_v4=8621 ru_v6=2186)` и `frr-reload completed` (или `frr.conf unchanged — skipping reload`).

### 5. firewalld-зона резолвится как ожидалось?

```bash
# в какой зоне живёт мой интерфейс?
firewall-cmd --get-zone-of-interface=vpns0
firewall-cmd --get-zone-of-interface=wg0
firewall-cmd --get-zone-of-interface=podman4

# не забытые rich-rules?
firewall-cmd --zone=trusted --list-rich-rules
firewall-cmd --zone=public  --list-rich-rules
```

Wildcard-интерфейсы (`vpns+`, `vpns*`) не показываются через `--get-zone-of-interface` — только через `--list-interfaces`.

### 6. Здоровье WireGuard

```bash
wg show wg0
# - handshake должен быть < 3 мин назад
# - счётчики transfer растут
# - allowed-ips покрывает 0.0.0.0/0, ::/0
```

Если handshake'и не завершаются — сниффай пакеты между двумя endpoint'ами:

```bash
tcpdump -nn -i ens1 udp port 31518
```

### 7. BGP

```bash
vtysh -c 'show bgp summary'
vtysh -c 'show bgp ipv4 unicast community 64512:201 | head'
vtysh -c 'show bgp neighbor fdf3:bb42:9fc6:ffff::3 advertised-routes'
ip -d route show table 10 proto bgp | head
```

`proto bgp` = `zebra (FRR)`. Если маршрут в BGP RIB, но не в kernel'е — проверь `ip protocol bgp route-map BGP-MAIN-FIB` — он может запрещать установку в `main` (намеренно для страновых префиксов, случайно — в остальных случаях).

## Частые ошибки ввода

| Ошибка                                                                  | Восстановление                                                          |
|-------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Забыл добавить BEGIN/END-маркеры в `frr.conf`                           | georoute упадёт с `errBeginMissing` — добавь маркеры, перепрогони       |
| Отсутствует WG `Table = off`                                            | wg-quick ставит default-маршруты, которые тебе не нужны — поставь `Table = off` |
| `vpns+` не в firewalld trusted                                          | ответы блокируются — `firewall-cmd --permanent --zone=trusted --add-interface=vpns+ && firewall-cmd --reload` |
| `connect-vrf.sh` без `chmod +x`                                         | ocserv молча пропускает его — клиенты работают, но VRF-изоляции нет     |
| У клиентов `ipv6-subnet-prefix=128` нет v6-маршрута в main              | гарантируется `ip -6 route replace … table 10` в connect-vrf.sh         |
| Sibling pool не в main FIB                                              | static-маршрут из `vrf-vpn.service` `sibling_v4_pool` / `sibling_v6_pool` |

## Откат плохого деплоя

```bash
cd /opt/project/repositories/infra-config
git log --oneline -10            # найди последний known-good коммит
git revert <bad-commit>          # создаёт обратный коммит
ansible-playbook playbooks/site.yml --check --diff
ansible-playbook playbooks/site.yml
```

Ручное состояние, добавленное вне playbook'а (например, разовый `ip route add`), этим **не** откатывается — это нужно отменять руками или кодифицировать в роль.

## Когда подозревать само ядро

Cross-VRF поведение хрупкое и имело upstream-регрессии:

- Серия "vrf: Reset skb conntrack connection on VRF rcv" — реверчена, потому что ломала DNAT для форвардящегося трафика.
- FRR issue [#15909](https://github.com/FRRouting/frr/issues/15909) — kernel 6.6/6.7 cross-VRF leak с багом TTL.

Если у тебя ядро `6.6.x` или `6.7.x` и DNAT через VRF ведёт себя нестабильно — апгрейдись на `6.8+` (или UEK R8 `6.12`).

## Что читать дальше

- [References](11-references.md) — первичная документация на каждый компонент.
