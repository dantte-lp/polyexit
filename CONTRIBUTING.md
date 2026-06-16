# Contributing to polyexit

## Reporting issues

Use the GitHub issue tracker. For security issues, see [SECURITY.md](SECURITY.md).

## Local development environment

```bash
git clone https://github.com/dantte-lp/polyexit.git
cd polyexit
dnf install -y ansible-core
ansible-galaxy collection install ansible.posix community.general
pip install ansible-lint
```

## Pull-request checklist

Before opening a PR:

1. `ansible-playbook --syntax-check playbooks/site.yml` passes.
2. `ansible-lint` clean (see `.ansible-lint` config if added).
3. If you touched a role, dry-run on at least one host:
   ```bash
   ansible-playbook playbooks/site.yml --check --diff --tags <role>
   ```
4. If you added a host variable, update `docs/en/04-inventory.md` and
   `docs/ru/04-inventory.md`.
5. If you added a role, add a section to `docs/en/05-roles.md` and
   `docs/ru/05-roles.md`.
6. Markdown changes: validate any new Mermaid diagrams with
   `@mermaid-js/mermaid-cli` (see [`docs/README.md`](docs/README.md)).
7. Commit messages: imperative mood, prefix with the area —
   `roles(vrf-vpn): …`, `docs(en): …`, `inventory: …`.

## Style

- **YAML** — 2-space indent; quote strings only when ambiguity is
  possible (IPv6 addresses, network names with colons, leading `*`).
- **Templates (`*.j2`)** — match the indentation style of the surrounding
  Jinja file; prefer `{% if %}` over inline conditionals for readability.
- **Shell** — `set -euo pipefail` at the top; `bin/` scripts target Bash
  4+; lint with `shellcheck`.
- **Docs** — declarative-imperative tone; no narrative softeners.
  See [`docs/README.md`](docs/README.md) for the doc conventions.

## Adding a new role

1. Create `roles/<name>/{tasks,handlers,templates}/main.yml`.
2. Add the role to `playbooks/site.yml` with appropriate `when:` and
   `tags:`.
3. Document the role in `docs/en/05-roles.md` and the RU translation.
4. Add idempotency proof: re-running the playbook reports `changed=0`.

## Adding a new host

See [`docs/en/04-inventory.md`](docs/en/04-inventory.md) (section "Adding a
new host") and [`docs/en/07-country-exit-bootstrap.md`](docs/en/07-country-exit-bootstrap.md)
for the bootstrap checklist.

## Reviewing

PR review is required before merge to `main`. The author cannot
self-approve. Conversations must be resolved before merge.
