# 06 — Deployment

## Controller prerequisites

```bash
# Oracle Linux 10 / Rocky 10 / Alma 10 / RHEL 10 controller
dnf install -y ansible-core git go
ansible-galaxy collection install ansible.posix community.general
git --version          # >= 2.40
ansible --version      # >= 2.16
go version             # >= 1.26 — for compiling the georoute binary in-tree
```

The controller is also a fleet member in this project — `dev-03` plays both roles. That is convenient but not required.

## Sanity check

```bash
cd /opt/project/repositories/infra-config
ansible-inventory --list                  # parses, no errors
ansible-playbook --syntax-check playbooks/site.yml
ansible all -m ping                       # all hosts reachable
```

## Dry-run with diff

```bash
ansible-playbook playbooks/site.yml --check --diff
```

This **does not** mutate any host. Read the diff carefully — Ansible will tell you every file it would change.

## First apply

```bash
ansible-playbook playbooks/site.yml
```

Expected duration: ~30 s for a host that is already mostly converged; ~3 min on a fresh box (the `common` role does a full `dnf install`).

## Selective applies

```bash
# one role only
ansible-playbook playbooks/site.yml --tags vrf-vpn

# multiple roles
ansible-playbook playbooks/site.yml --tags vrf-vpn,nft-vpn

# one host only
ansible-playbook playbooks/site.yml --limit dev-04

# one host + one role
ansible-playbook playbooks/site.yml --limit dev-05 --tags georoute
```

Tags map 1:1 to role names (see [`playbooks/site.yml`](../../playbooks/site.yml)).

## Output verbosity

```bash
ansible-playbook playbooks/site.yml -v       # show task names + skipping reasons
ansible-playbook playbooks/site.yml -vv      # + module args
ansible-playbook playbooks/site.yml -vvv     # + SSH transcript
ansible-playbook playbooks/site.yml -vvvv    # + connection plugin debug
```

## Vault flow

```bash
# first time
echo 'YOUR_VAULT_PASSWORD' > ~/.ansible_vault_pass
chmod 600 ~/.ansible_vault_pass

# edit ansible.cfg to uncomment the vault_password_file line:
#   vault_password_file = ~/.ansible_vault_pass

cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml
$EDITOR inventory/group_vars/vault.yml
ansible-vault encrypt inventory/group_vars/vault.yml

# later — change a secret
ansible-vault edit inventory/group_vars/vault.yml
```

Reference: [ansible-vault docs](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

## What ansible-playbook does NOT manage

This needs to be set up by hand once:

| Concern                          | Status                                                                  |
|----------------------------------|-------------------------------------------------------------------------|
| WireGuard tunnels (`wg*` interfaces) | manual (Ansible role for this is a planned addition)                |
| FRR `/etc/frr/frr.conf`          | manual — Ansible role planned for full BGP mesh generation              |
| TLS certificates                 | issued via `certbot` on `dev-03` and `rsync`ed to siblings via deploy-hook |
| `ocpasswd` initial seed          | not managed; use [`bin/vpn-user add`](08-user-management.md) to populate |
| `chrony` config                  | only the package is installed — pin your pool yourself                  |
| SELinux                          | left in whatever state the OS image has                                 |

## Idempotency

The playbook is fully idempotent. A second run with no inventory changes prints:

```text
PLAY RECAP *********************************************************************
dev-03  : ok=18  changed=0  unreachable=0  failed=0  skipped=5  rescued=0  ignored=0
dev-04  : ok=18  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

The `changed=0` is the contract. Any non-zero number is a real change — review with `--diff`.

## Roll-forward strategy

Apply changes incrementally:

```bash
# 1. dry-run on one host
ansible-playbook playbooks/site.yml --limit dev-04 --check --diff

# 2. apply on one host
ansible-playbook playbooks/site.yml --limit dev-04

# 3. validate from a real client (don't trust server-local tests for tunnel paths)
#    see docs/en/10-troubleshooting.md for the validation script

# 4. apply to the rest
ansible-playbook playbooks/site.yml
```

## Roll-back strategy

This repo is a git repo; revert is `git revert <commit>`, then re-apply. Ansible re-renders templates to the previous content and reverses any service-level changes through handler re-triggering. State that lives **outside** the playbook (e.g. firewalld rich-rules added manually, manual `ip route` adds) is not reverted automatically.

## What to read next

- [Adding a country exit](07-country-exit-bootstrap.md) — production bring-up.
- [User management](08-user-management.md) — adding VPN users.
