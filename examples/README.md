# polyexit — example inventory and playbook

These files are a sanitized starting point. Real production state
lives in a private repository — this folder is the public, working
template you can fork into your own private repo and fill in.

## Usage

```bash
# 1. fork dantte-lp/polyexit into your own private repo
# 2. copy the example tree to the repo root
cp -r examples/ansible.cfg examples/inventory examples/playbooks .

# 3. fill in real values
$EDITOR inventory/host_vars/dev-NN/00-base.yml.example
mv     inventory/host_vars/dev-NN/00-base.yml.example \
       inventory/host_vars/dev-NN/00-base.yml

# 4. encrypt vault
cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml
$EDITOR inventory/group_vars/vault.yml
ansible-vault encrypt inventory/group_vars/vault.yml

# 5. dry-run
ansible-playbook playbooks/site.yml.example --check
```

## Conventions

- All IPv4 addresses use RFC 5737 documentation prefixes:
  192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24.
- All IPv6 addresses use RFC 3849 (2001:db8::/32).
- All ASNs are the RFC 6996 private range (4200000000–4294967294).
- See [`../docs/`](../docs/) for the architectural narrative.
