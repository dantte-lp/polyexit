# ocserv-fleet — documentation

Ansible-managed fleet of OpenConnect (`ocserv`) VPN exit nodes with per-country geo-routing, isolated client traffic via Linux VRF, BGP-connected sibling sites, and atomic prefix-feed updates.

## Languages

| Language | Status        | Path             |
|----------|---------------|------------------|
| English  | canonical     | [`en/`](en/)     |
| Русский  | translation   | [`ru/`](ru/)     |

## Navigation (English)

| # | Section                                                                   | Read when                                            |
|---|---------------------------------------------------------------------------|------------------------------------------------------|
| 0 | [How it all fits](en/00-how-it-works.md)                                  | **start here** — worked example with real configs    |
| 1 | [Overview](en/01-overview.md)                                             | first contact — what the project does and why        |
| 2 | [Architecture](en/02-architecture.md)                                     | needing a mental model of data and control planes    |
| 3 | [Environment](en/03-environment.md)                                       | provisioning a new host — OS, kernel, packages       |
| 4 | [Inventory](en/04-inventory.md)                                           | adding hosts, changing per-host values               |
| 5 | [Roles](en/05-roles.md)                                                   | understanding what every Ansible role does           |
| 6 | [Deployment](en/06-deployment.md)                                         | running the playbook for the first time              |
| 7 | [Adding a country exit](en/07-country-exit-bootstrap.md)                  | bringing up a new country-exit node end-to-end       |
| 8 | [User management](en/08-user-management.md)                               | adding, removing, auditing VPN users                 |
| 9 | [georoute](en/09-georoute.md)                                             | the Go binary that feeds country prefixes to FRR+nft |
| 10 | [Troubleshooting](en/10-troubleshooting.md)                              | a path is broken; which file / counter to check      |
| 11 | [References](en/11-references.md)                                        | upstream documentation for every component           |

## Navigation (Русский)

| # | Раздел                                                                    | Когда читать                                         |
|---|---------------------------------------------------------------------------|------------------------------------------------------|
| 0 | [Как всё стыкуется](ru/00-how-it-works.md)                                | **начни тут** — рабочий пример с реальными конфигами |
| 1 | [Обзор](ru/01-overview.md)                                                | первое знакомство с проектом                         |
| 2 | [Архитектура](ru/02-architecture.md)                                      | модель data plane и control plane                    |
| 3 | [Окружение](ru/03-environment.md)                                         | provisioning нового хоста                            |
| 4 | [Инвентарь](ru/04-inventory.md)                                           | добавление хостов, host_vars                         |
| 5 | [Роли](ru/05-roles.md)                                                    | что делает каждая Ansible-роль                       |
| 6 | [Развёртывание](ru/06-deployment.md)                                      | первый прогон playbook                               |
| 7 | [Добавление country-exit](ru/07-country-exit-bootstrap.md)                | подъём нового странового exit-узла                   |
| 8 | [Управление пользователями](ru/08-user-management.md)                     | add / del / list VPN-юзеров                          |
| 9 | [georoute](ru/09-georoute.md)                                             | Go-бинарник для country prefix → FRR + nft           |
| 10 | [Troubleshooting](ru/10-troubleshooting.md)                              | путь сломан; куда смотреть                           |
| 11 | [Ссылки](ru/11-references.md)                                            | upstream-документация для каждого компонента         |

## Diagram conventions

All diagrams use [Mermaid](https://mermaid.js.org/). GitHub renders them natively. To render locally:

```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i diagram.md -o diagram.svg
```

## Code-block conventions

- `bash` blocks — shell commands meant to be run as `root` unless prefixed with `$` (then a normal user).
- `yaml` blocks — Ansible inventory / vars / task files.
- `nft` blocks — nftables rule files.
- `go` blocks — Go source.

Every command is intentionally **copy-pasteable** — no `<placeholder>` syntax inside command lines (placeholders live in YAML files instead).
