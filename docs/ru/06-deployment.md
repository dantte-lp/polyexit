# 06 — Развёртывание

## Требования к контроллеру

```bash
# Oracle Linux 10 / Rocky 10 / Alma 10 / RHEL 10 контроллер
dnf install -y ansible-core git go
ansible-galaxy collection install ansible.posix community.general
git --version          # >= 2.40
ansible --version      # >= 2.16
go version             # >= 1.26 — для компиляции бинаря georoute в дереве
```

Контроллер в этом проекте — также член флота — `dev-03` играет обе роли. Удобно, но не обязательно.

## Sanity check

```bash
cd /opt/project/repositories/infra-config
ansible-inventory --list                  # парсится, без ошибок
ansible-playbook --syntax-check playbooks/site.yml
ansible all -m ping                       # все узлы достижимы
```

## Dry-run с diff

```bash
ansible-playbook playbooks/site.yml --check --diff
```

Это **не** мутирует ни один узел. Внимательно читай diff — Ansible покажет каждый файл, который изменил бы.

## Первое применение

```bash
ansible-playbook playbooks/site.yml
```

Ожидаемая длительность: ~30 с для узла, уже почти сошедшегося; ~3 мин на свежей машине (роль `common` делает полный `dnf install`).

## Селективные применения

```bash
# только одна роль
ansible-playbook playbooks/site.yml --tags vrf-vpn

# несколько ролей
ansible-playbook playbooks/site.yml --tags vrf-vpn,nft-vpn

# только один узел
ansible-playbook playbooks/site.yml --limit dev-04

# один узел + одна роль
ansible-playbook playbooks/site.yml --limit dev-05 --tags georoute
```

Теги мапятся 1:1 на имена ролей (смотри [`playbooks/site.yml`](../../playbooks/site.yml)).

## Уровень логирования

```bash
ansible-playbook playbooks/site.yml -v       # имена тасков + причины skip
ansible-playbook playbooks/site.yml -vv      # + аргументы модулей
ansible-playbook playbooks/site.yml -vvv     # + SSH-транскрипт
ansible-playbook playbooks/site.yml -vvvv    # + debug connection-плагина
```

## Vault flow

```bash
# первый раз
echo 'YOUR_VAULT_PASSWORD' > ~/.ansible_vault_pass
chmod 600 ~/.ansible_vault_pass

# в ansible.cfg раскомментируй строку vault_password_file:
#   vault_password_file = ~/.ansible_vault_pass

cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml
$EDITOR inventory/group_vars/vault.yml
ansible-vault encrypt inventory/group_vars/vault.yml

# позже — поменять секрет
ansible-vault edit inventory/group_vars/vault.yml
```

Справка: [ansible-vault docs](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

## Что ansible-playbook НЕ управляет

Это нужно настроить руками один раз:

| Компонент                        | Статус                                                                  |
|----------------------------------|-------------------------------------------------------------------------|
| WireGuard-туннели (`wg*` интерфейсы) | вручную (Ansible-роль для этого запланирована)                      |
| FRR `/etc/frr/frr.conf`          | вручную — Ansible-роль запланирована для генерации полного BGP-меша     |
| TLS-сертификаты                  | выпускаются `certbot`'ом на `dev-03` и `rsync`ятся соседям через deploy-hook |
| Начальный `ocpasswd`             | не управляется; используй [`bin/vpn-user add`](08-user-management.md) для наполнения |
| Конфиг `chrony`                  | устанавливается только пакет — пин пула делай сам                       |
| SELinux                          | оставлен в том состоянии, в котором его выдал OS-образ                  |

## Идемпотентность

Playbook полностью идемпотентен. Второй прогон без изменений инвентаря печатает:

```text
PLAY RECAP *********************************************************************
dev-03  : ok=18  changed=0  unreachable=0  failed=0  skipped=5  rescued=0  ignored=0
dev-04  : ok=18  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

`changed=0` — это контракт. Любое ненулевое число — это реальное изменение — смотри `--diff`.

## Стратегия roll-forward

Применяй изменения инкрементально:

```bash
# 1. dry-run на одном узле
ansible-playbook playbooks/site.yml --limit dev-04 --check --diff

# 2. применить на одном узле
ansible-playbook playbooks/site.yml --limit dev-04

# 3. провалидировать с реального клиента (не доверяй server-local тестам туннельных путей)
#    смотри docs/ru/10-troubleshooting.md для скрипта валидации

# 4. применить на остальном
ansible-playbook playbooks/site.yml
```

## Стратегия roll-back

Этот репозиторий — git-репозиторий; revert — это `git revert <commit>`, затем re-apply. Ansible перерендерит шаблоны в предыдущее содержимое и развернёт сервис-уровневые изменения через повторное срабатывание handler'ов. Состояние, живущее **вне** playbook'а (например, firewalld rich-rules, добавленные руками, ручные `ip route` add), автоматически не откатывается.

## Что читать дальше

- [Добавление country exit](07-country-exit-bootstrap.md) — production-ввод.
- [Управление пользователями](08-user-management.md) — добавление VPN-пользователей.
