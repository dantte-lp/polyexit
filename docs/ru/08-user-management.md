# 08 — Управление пользователями

VPN-пользователи управляются ocserv через утилиту `ocpasswd`. Флот держит `/etc/ocserv/ocpasswd` на каждом узле в синхронизации **прогоном `ocpasswd` на каждом узле через Ansible**, а не rsync'ом единого золотого файла. Каждый узел генерирует собственную соль для одного и того же пароля — итоговые файлы отличаются побайтно, но `ocserv` валидирует относительно любого из них.

## Обёртка: `bin/vpn-user`

```bash
bin/vpn-user add <user>                  # запрашивает пароль (read -s)
bin/vpn-user add <user> <password>       # one-shot, следи за shell history
bin/vpn-user del <user>
bin/vpn-user list
```

Источник: [`bin/vpn-user`](../../bin/vpn-user)

Обёртка — тонкий шелл поверх `ansible-playbook playbooks/manage-user.yml -e vpn_action=...`. Используй playbook напрямую, если нужны дополнительные Ansible-флаги:

```bash
ansible-playbook playbooks/manage-user.yml \
    -e vpn_action=add \
    -e vpn_user=alice \
    -e vpn_pass='generated-password-here' \
    --limit exits-country-local
```

## Примеры

```bash
# Добавить пользователя, пароль интерактивно
$ bin/vpn-user add alice
Password for alice: ********
Confirm password:      ********

# Добавить с явным паролем (например, скриптом)
$ bin/vpn-user add bob 'OneTimeP@ss12'

# Удалить
$ bin/vpn-user del bob

# Аудит — одинаковый список пользователей на каждом узле?
$ bin/vpn-user list
…
ok: [dev-03] => msg: 'dev-03 (11 users): alice, bmv, …'
ok: [dev-04] => msg: 'dev-04 (11 users): alice, bmv, …'
```

Если списки расходятся — это баг — найди, какой узел отстал, и перепрогони add для недостающего пользователя.

## Генерация пароля

```bash
# 20-символьный base64-алфавит, без `/+=`
openssl rand -base64 18 | tr -d '/+=' | head -c 20
```

Сохрани вывод в свой password manager **прежде**, чем вызывать `bin/vpn-user add` — как только playbook прошёл задачу с `no_log: true`, значение уже не в логах.

## Поведение при успехе

- Каждый достижимый узел получает добавленного одного и того же пользователя.
- Handler `occtl reload` срабатывает на каждом изменённом узле — **graceful**; активные сессии не терминируются, и push'нутый им конфиг не меняется. Новая сессия = новый push.
- Placeholder-узлы (например, `dev-05` с `ansible_host: "{{ public_v4 }}"`, где `public_v4: TBD`) репортят `unreachable` — это ожидаемо и не повод реагировать.

## Массовый импорт пользователей

```bash
while IFS=: read -r user pass; do
    bin/vpn-user add "$user" "$pass"
done < users.tsv
```

Формат `users.tsv`: `user:password` на строку. Следи за shell history — `set +o history` перед циклом.

## Модель аутентификации

Этот playbook использует backend `plain` ocserv (`/etc/ocserv/ocpasswd`). Альтернативы, работающие с тем же флотом, но требующие подмены шаблона `ocserv.conf`:

| Backend             | Когда использовать                                                           |
|---------------------|------------------------------------------------------------------------------|
| `plain` (default)   | малые флоты, без IdP                                                          |
| `pam`               | синхронизация с системными пользователями; интеграция с FreeIPA / SSSD        |
| `radius`            | корпоративный RADIUS-сервер (FreeRADIUS, NPS)                                 |
| `oidc`              | новые сборки ocserv; SSO через Keycloak / Authentik                           |

Справка: [`ocserv.8` — секция Authentication](https://ocserv.openconnect-vpn.net/ocserv.8.html#sect3).

## Что читать дальше

- [georoute](09-georoute.md) — взгляд оператора на префикс-фид.
- [Troubleshooting](10-troubleshooting.md) — когда логин или маршрутизация ломаются.
